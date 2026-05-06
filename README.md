# Silo

<!-- allow HTML in Markdown for explicit formatting e.g., line breaks: -->
<!-- markdownlint-disable MD033 -->

Silo is a Golang library for managing persistent containers.

All behaviors are independent of the calling process:

- "Process A" may start a container
- "Process B" may subscribe to events from the container
- "Process C" may send `stdin` events to the container
- "Process D" may stop the container

Processes A, B, C, and D may be the same or different.

## Events

A silo isolates a process, its environment, dependencies, etc. from the host
system. All events are recorded in-order for each contained process:

- Output events
  - Silo start
    - Program path
    - Arguments
    - Environment
  - `stdout` and `stderr` output
    - Stream type (`stdout` or `stderr`)
    - Message content
  - Silo stop
    - Exit code
    - Signal (if applicable)

- Input events
  - Messages sent to silo `stdin`
    - Provider PID
    - Message content

- Control events
  - Silo stop
    - Provider PID
  - Silo purge
    - Provider PID

All events are timestamped and assigned a unique ID as received by the `silod`
daemon, then recorded to disk. Event ID is a monotonically increasing integer,
starting at 1 for the first event. Each silo maintains its own event history,
meaning each silo has its own event ID sequence starting at 1.

New subscribers receive full event history up to the current moment (if
requested), then receive new events as they occur. This allows subscribers to
view a silo's complete event history even after the process has stopped.

Sending invalid input (e.g. `stdin` event when the silo is stopped) returns an
error to the provider, and does not affect the silo's event history or state.

All events are subscribable: a process subscribed to `stdin` events that sends a
message to `stdin` will receive an event for that message, listing its own PID
as the provider. This allows processes to observe their own interactions with
the silo, as well as those of other processes.

Event history is stored on disk in a JSON-formatted file per silo. History is
retained until calling `silo.Purge()`.

Calling `silo.Stop()` on a running silo stops the contained process but does not
unregister subscribers or clear event history. Stop method is configurable:

- `silo.TERM` (default): sends `SIGTERM` to the contained process, allowing it
  to exit gracefully. If the process does not exit within a configurable
  timeout, it is forcefully killed with `SIGKILL`.

- `silo.CLOSE`: closes the silo's `stdin` stream, allowing the process to exit
  gracefully. If the process does not exit within a configurable timeout, stop
  method falls back to `silo.TERM`.

- `silo.KILL`: sends `SIGKILL` to the contained process, stopping it
  immediately.

Calling `silo.Purge()` on a stopped silo will notify & unregister all
subscribers, then destroy all event history for the specified ID.

A process may subscribe to any silo ID, even ones which do not exist. They will
receive any available event history, then new events as they occur. This allows
subscription to silos before or after they exist, and ensures a complete event
history beginning with the first start event.

## Daemon

Daemon process `silod` maintains silo interaction. Only one instance runs at a
time. Automatically starts, if necessary, during calls like `silo.New()` or
`silo.Subscribe()`.

Each subscriber communicates over its own socket connection to the daemon.

Daemon spawns a new thread per subscriber. The "subscriber thread" is
responsible for:

- Subscriber socket connection
- Queue silo events to write to the subscriber (socket)
  - Queue allows each subscriber to read events at its own pace, without
    blocking the silo or other subscribers
  - If requested, queue silo event history before new events
- Queue subscriber input events to write to the silo
  - Re-published by silo as `stdin` event after successful receipt
  - Queue prevents overwhelming the silo with too many messages at once
- Cleanup/close when the subscriber socket closes

Daemon spawns a new thread per silo. The "silo thread" is responsible for:

- Silo events
  - Record events to history on disk, then write to container (`stdin` events)
    and/or publish to subscribers (all events)
- Silo lifecycle
  - Publish start event after successful start
  - Publish stop event after successful stop
  - Publish purge event after successful stop, before unregistering subscribers
- Silo history file
  - Each line contains one event encoded as a JSON object
  - Recorded in receipt order; the event ID of line N+1 is always greater than
    the event ID of line N

## Technology

- Written in `Go`
  - [Homepage](https://go.dev)
- Docker for image & high-level container management
  - [Docker](https://www.docker.com)
- `containerd` for low-level container management
  - [Homepage](https://containerd.io)
  - [GitHub](https://github.com/containerd/containerd)

## Create a Silo

```go
import "github.com/dstodev/silo"

func main() {
    s, err := silo.New(
        "my-unique-silo-id",
        FromDockerImage("alpine:latest"),
    )
    s.Run([]string{"echo", "Hello, world!"})
}
```

## Subscribe to Events

```go
func myEventHandler(event silo.Event) {
    switch e := event.(type) {
    case silo.EventStart:
        fmt.Printf("Start %s\n", e.SiloID)
    case silo.EventInput:
        fmt.Printf("Process %d sent: %s\n", e.ProviderPID, e.MessageContent)
    case silo.EventOutput:
        fmt.Printf("[%s] %s\n", e.StreamName, e.MessageContent)
    case silo.EventStop:
        fmt.Printf("Stop %s (%d)\n", e.SiloID, e.ExitCode)
    }
}

func main() {
    silo.Subscribe("my-unique-silo-id", myEventHandler)
}
```
