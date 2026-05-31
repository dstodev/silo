# Daemon

<!-- markdownlint-disable MD033 -->

[← README](../README.md)

Daemon process `silod` maintains silo abstraction. Only one instance runs at a
time. The daemon automatically starts, if necessary, during calls like
`silo.Run()` or `silo.Subscribe()`. The first process to bind the socket path
starts the daemon; others wait and connect once it is ready. If a stale socket
remains from a prior crash, it is removed before rebinding.

Each new run of the daemon creates a new workspace directory:

- `$XDG_RUNTIME_DIR/silo/<daemon-pid>/` if available
- `/tmp/silo/<daemon-pid>/` otherwise

Environment variable `$SILO_WORKSPACE` contains the selected workspace
directory.

The daemon listens on a Unix domain socket in the `silo` workspace directory:

- `$SILO_WORKSPACE/silod.sock`

The socket file permissions restrict access to the owning user only.

The workspace holds one history file per silo. A new daemon instance has no
knowledge of any previous instances or their silos.

The daemon starts a new goroutine per subscriber. Each "subscriber goroutine" is
responsible for:

- Subscriber socket connection
- Accumulate events to send to the subscriber
  - Receives events via unbounded in-process queue from silo goroutines
- Queue silo events to write to the subscriber (socket)
  - Because each subscriber has its own goroutine, a slow subscriber does not
    block the silo or other subscribers
  - If the socket buffer is full (i.e. the subscriber is not draining fast
    enough), the goroutine waits and logs a warning.
  - If requested, queue silo event history before new events
- Queue subscriber input events to write to the silo
  - Re-published as `event.MessageFromProvider` after successful receipt
  - Queue prevents overwhelming the silo with too many messages at once
- Cleanup/close when the subscriber socket closes (triggered by `sub.Cancel()`)

On the client side, the library drains events from the socket into an unbounded
in-process queue; growth is bounded only by available client process memory.

The daemon starts a new goroutine per `silo.Run` (container). The "silo
goroutine" is responsible for:

- Silo events
  - Receive user events from subscriber goroutines
  - Receive container events from the container runtime & the container's
    `stdout` and `stderr` streams
  - For each received event, in order of receipt:
    - Record to the silo history file
    - Publish to subscriber goroutines
- Silo lifecycle
  - Publish `event.ContainerStarted` after successful start
  - Publish `event.ContainerStopped` after the contained process exits
  - Publish `event.SiloPurged` before unregistering subscribers
- Silo history file
  - Stored in $SILO_WORKSPACE
  - Each line contains one event encoded as a JSON object
  - Recorded in receipt order; line N+1 event ID > line N event ID
