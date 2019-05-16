# The hostcall API

The hostcall service is managed by one or more consumer threads that
process "packets" received from active kernels.

    hostcall_consumer_t *consumer = hostcall_create_consumer(...);
    hostcall_launch_consumer(consumer);

The packets are submitted by active kernels in "hostcall buffers"
registered with the consumer. The client must associate one hostcall
buffer with each queue on the device. Behaviour is undefined if no
buffer is associated with a queue, or the same buffer is associated
with multiple queues.

For independent forward progress of each wave on a queue, the
corresponding buffer must have at least as many packets as the number
of waves that may be launched on that queue.

    size_t size = hostcall_get_buffer_size(num_packets);
    uint32_t alignment = hostcall_get_buffer_alignment();
    void *buffer = allocate_buffer(size, alignment);
    hostcall_initialize_buffer(buffer, size, num_packets);

Buffers may be registered even with a running consumer.

    hostcall_register_buffer(consumer, buffer);

The client may launch multiple consumers for additional
throughput. Behaviour is undefined if the same buffer is registered
with multiple consumers.

The client introduces a service on the hostcall infrastructure by
registering a service handler function with a numeric service ID. The
space of service IDs must be managed by the client. Attempting to
register a new service handler with an existing service ID will
over-write the previously registered handler. Service ID 0 is reserved
for registering a default handler.

Each packet arriving in a hostcall buffer specifies a service ID. The
consumer attempts to invoke the handler registered for that service
ID. If no handler is found, the consumer attempts to invoke the
default handler instead. If no handler is found, the consumer causes
the application to halt.

    void *cbdata; // additional data passed by client
    handler();    // function of type hostcall_service_handler_t
    hostcall_register_service(consumer, service_id, handler, cbdata);

All registered buffers are owned by the client and never destroyed by
the consumer. When a buffer is no longer needed, the client may
deregister it. The consumer thread safely ignores a deregistered buffer,
and the client may destroy the buffer at any point after deregistering.

    hostcall_deregister_buffer(consumer, buffer);

The client eventually destroys a consumer when its services are no
longer needed.

    hostcall_destroy_consumer(consumer);

# Build instructions

```
mkdir build
cd build
cmake -DCMAKE_INSTALL_PREFIX=/path/to/install ..
make
make build-tests
make test
make install
```
