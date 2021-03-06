##===----------------------------------------------------------------------===##
#
#                     The LLVM Compiler Infrastructure
#
# This file is dual licensed under the MIT and the University of Illinois Open
# Source Licenses. See LICENSE.txt for details.
##===----------------------------------------------------------------------===##
#
# aomp-extras/hostcall/libdevice/CMakeLists.txt
# Written by Greg Rodgers (Gregory.Rodgers@amd.com)
#
##===----------------------------------------------------------------------===##

cmake_minimum_required(VERSION 3.0 FATAL_ERROR)
if("${CMAKE_SOURCE_DIR}" STREQUAL "${CMAKE_CURRENT_SOURCE_DIR}")
  project(hostcall-project)
  message("--------------------------------------------")
endif()

# find the llvm compiler
find_package(LLVM QUIET CONFIG
  PATHS
  $ENV{AOMP}
  $ENV{HOME}/rocm/aomp
  /opt/rocm/aomp
  /usr/lib/rocm/aomp
  ${CMAKE_CXX_COMPILER_DIR}
  NO_DEFAULT_PATH
)

if (LLVM_DIR)
  message("   -- Building hostcall libdevice with LLVM ${LLVM_PACKAGE_VERSION} found at ${LLVM_INSTALL_PREFIX}")
else()
  message("   ERROR: NO LLVM FOUND! Not building hostcall libdevice.")
  return()
endif()

# Assome rocm-device-libs repository is next to aomp-extras repository
set(ROCDL ${CMAKE_CURRENT_SOURCE_DIR}/../../../rocm-device-libs)
set(ROCDL_INC_OCKL ${ROCDL}/ockl/inc)
set(ROCDL_INC_OCML ${ROCDL}/ocml/inc)
set(ROCDL_INC_IRIF ${ROCDL}/irif/inc)

set(HIPINC ${CMAKE_CURRENT_SOURCE_DIR}/../../../hip/include)

# add_custom_target(hostcall ALL)

#optimization level
set(optimization_level 2)

# Only list the sources we want to compile. Do not list needs_hostcall_buffer.hip
# which is compiled and linked separately below.
set(sources
  ${CMAKE_CURRENT_SOURCE_DIR}/src/hostcall_stubs.hip
  ${CMAKE_CURRENT_SOURCE_DIR}/src/hostcall_invoke.cl)


# for both in-tree and out-of-tree build
if (NOT CMAKE_ARCHIVE_OUTPUT_DIRECTORY)
  set(OUTPUTDIR ${CMAKE_CURRENT_BINARY_DIR})
else()
  set(OUTPUTDIR ${CMAKE_ARCHIVE_OUTPUT_DIRECTORY})
endif()

# Get list of AMD GPUs to build for 
set(mcpus $ENV{GFXLIST})
if(mcpus)
  message("   -- Building for devices specified by GFXLIST: ${mcpus}")
else()
  set(mcpus gfx700 gfx701 gfx801 gfx803 gfx900)
  message("   -- Building default set of AMD GPUs: ${mcpus}")
  message("   -- You may override default with GFXLIST environment variable ")
endif()
separate_arguments(mcpus)

macro(collect_sources name dir)
  set(hip_sources)
  set(ocl_sources)
  set(llvm_sources)
  foreach(file ${ARGN})
    file(RELATIVE_PATH rfile ${dir} ${file})
    get_filename_component(rdir ${rfile} DIRECTORY)
    get_filename_component(fname ${rfile} NAME_WE)
    get_filename_component(fext ${rfile} EXT)
    #file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${rdir})
    if (fext STREQUAL ".hip")
      set(cfile ${CMAKE_CURRENT_BINARY_DIR}/${rdir}/${fname}.hip)
      list(APPEND hip_sources ${cfile})
    endif()
    if (fext STREQUAL ".cl")
      set(cfile ${CMAKE_CURRENT_BINARY_DIR}/${rdir}/${fname}.cl)
      list(APPEND ocl_sources ${cfile})
    endif()
    if (fext STREQUAL ".ll")
      list(APPEND csources ${file})
      set(cfile ${CMAKE_CURRENT_BINARY_DIR}/${rdir}/${fname}.ll)
      list(APPEND llvm_sources ${cfile})
    endif()
  endforeach()
endmacro()

macro(add_hip_bc_library name dir)

  set(hip_cmd ${LLVM_INSTALL_PREFIX}/bin/clang++
    -x hip
    -std=c++11
    -fcuda-rdc
    -fvisibility=default
    --cuda-device-only
    -Wno-unused-value
    -O${optimization_level}
    --cuda-gpu-arch=${mcpu}
    -I${CMAKE_CURRENT_SOURCE_DIR}/../include
    -I${CMAKE_CURRENT_SOURCE_DIR}/src
    -I${LLVM_LIBRARY_DIRS}/libdevice/include
    -I${HIPINC})

#    --hip-auto-headers=cuda_open
#    --hip-auto-headers=cuda
  set(bc1_files)

  foreach(file ${ARGN})
    file(RELATIVE_PATH rfile ${dir} ${file})
    get_filename_component(rdir ${rfile} DIRECTORY)
    get_filename_component(fname ${rfile} NAME_WE)
    get_filename_component(fext ${rfile} EXT)

    set(bc1_filename ${fname}.${mcpu}.bc)

    file(GLOB h_files "${CMAKE_CURRENT_SOURCE_DIR}/../include/*.h")

    add_custom_command(
      OUTPUT ${bc1_filename}
      COMMAND ${hip_cmd} ${CMAKE_CURRENT_SOURCE_DIR}/src/${fname}.hip -o ${bc1_filename}
      DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/src/${fname}.hip" ${h_files}
      )

    list(APPEND bc1_files ${bc1_filename})
  endforeach()

  add_custom_command(
    OUTPUT linkout.hip.${mcpu}.bc
    COMMAND ${LLVM_INSTALL_PREFIX}/bin/llvm-link ${bc1_files} -o linkout.hip.${mcpu}.bc
    DEPENDS ${bc1_files}
    )

  list(APPEND bc_files linkout.hip.${mcpu}.bc)
endmacro()

macro(add_ocl_bc_library name dir)
	set(cl_cmd ${LLVM_INSTALL_PREFIX}/bin/clang
    -S -emit-llvm
    -DCL_VERSION_2_0=200 -D__OPENCL_C_VERSION__=200
    -Dcl_khr_fp64 -Dcl_khr_fp16
    -Dcl_khr_subgroups -Dcl_khr_int64_base_atomics -Dcl_khr_int64_extended_atomics
    -x cl -Xclang -cl-std=CL2.0 -Xclang -finclude-default-header
    -target amdgcn-amd-amdhsa
    -I${ROCDL_INC_OCKL}
    -I${ROCDL_INC_OCML}
    -I${ROCDL_INC_IRIF}
    -I${CMAKE_CURRENT_SOURCE_DIR}/src
    -I${LLVM_COMPILER}/include)
   set(cl_bc_files)
   file(GLOB h_files "${CMAKE_CURRENT_SOURCE_DIR}/../include/*.h")

   foreach(file ${ARGN})
     file(RELATIVE_PATH rfile ${dir} ${file})
     get_filename_component(fname ${rfile} NAME_WE)
     set(bc_filename ${fname}.bc)

     add_custom_command(
       OUTPUT ${bc_filename}
       COMMAND ${cl_cmd} ${CMAKE_CURRENT_SOURCE_DIR}/src/${fname}.cl -o ${bc_filename}
       DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/src/${fname}.cl ${h_files}
       )
     list(APPEND cl_bc_files ${bc_filename})
   endforeach()

  add_custom_command(
    OUTPUT linkout.ocl.${mcpu}.bc
    COMMAND ${LLVM_INSTALL_PREFIX}/bin/llvm-link ${cl_bc_files} -o linkout.ocl.${mcpu}.bc
    DEPENDS ${cl_bc_files}
    )

  list(APPEND bc_files linkout.ocl.${mcpu}.bc)
endmacro()

set(libname "hostcall-amdgcn")
collect_sources(${libname} ${CMAKE_CURRENT_SOURCE_DIR} ${sources})

foreach(mcpu ${mcpus})
  set(bc_files)
  add_hip_bc_library(${libname} ${CMAKE_CURRENT_SOURCE_DIR} ${hip_sources})
  add_ocl_bc_library(${libname} ${CMAKE_CURRENT_SOURCE_DIR} ${ocl_sources})
  add_custom_command(
    OUTPUT lib${libname}-${mcpu}.bc
    COMMAND ${LLVM_INSTALL_PREFIX}/bin/llvm-link ${bc_files} -o ${OUTPUTDIR}/lib${libname}-${mcpu}.bc
    DEPENDS ${bc_files}
    )
  add_custom_target(lib${libname}-${mcpu} ALL DEPENDS lib${libname}-${mcpu}.bc)
  install(FILES ${OUTPUTDIR}/lib${libname}-${mcpu}.bc 
     DESTINATION "lib${OPENMP_LIBDIR_SUFFIX}/libdevice"
  )
endforeach()

# Build needs_hostcall_buffer-amdgcn.bc. This bc is linked only when host services are needed.
# It should not be linked to the SDL libhostcall*.bc SDLs because then it would appear as if all
# applications need hostcall services.
add_custom_command(
  OUTPUT ${OUTPUTDIR}/needs_hostcall_buffer-amdgcn.bc
  COMMAND ${hip_cmd} ${CMAKE_CURRENT_SOURCE_DIR}/src/needs_hostcall_buffer.hip -o ${OUTPUTDIR}/needs_hostcall_buffer-amdgcn.bc
  DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/src/needs_hostcall_buffer.hip"
)
add_custom_target(needs_hostcall_buffer-amdgcn ALL DEPENDS ${OUTPUTDIR}/needs_hostcall_buffer-amdgcn.bc)
install(FILES ${OUTPUTDIR}/needs_hostcall_buffer-amdgcn.bc
  DESTINATION "lib${OPENMP_LIBDIR_SUFFIX}/libdevice"
)

install(FILES
    "${CMAKE_CURRENT_SOURCE_DIR}/../include/hostcall_service_id.h"
    "${CMAKE_CURRENT_SOURCE_DIR}/../include/hostcall_stubs.h"
    DESTINATION include)

if("${CMAKE_SOURCE_DIR}" STREQUAL "${CMAKE_CURRENT_SOURCE_DIR}")
  message("------ DONE libdevice hostcall cmake -------")
endif()
