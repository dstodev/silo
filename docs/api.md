# Silo API Reference

<!-- markdownlint-disable MD033 -->

[← README](../README.md)

This document describes the Silo API.

## Functions

### `silo.Run(id, options...)`

Run a program in a silo.

Silo IDs must match the regular expression `[a-zA-Z0-9_.+-]+` and may not exceed
200 characters. Returns `silo.InvalidArguments` if the ID is invalid.

`silo.Run()` accepts options:

- `silo.FromDockerImage()`: Use a pre-built Docker image e.g. `alpine:latest`.
  Incompatible with `silo.FromDockerfile()`.
- `silo.FromDockerfile()`: (Re)build and use an image from a Dockerfile by path.
  Incompatible with `silo.FromDockerImage()`.
- `silo.WithArgs()`: Start the contained program with additional arguments.
- `silo.WithMount()`: Mount a directory from the host to the container. Call
  multiple times to mount multiple directories.

Exactly one container image source is required.

Supplying incompatible options returns `silo.InvalidArguments`.

```go
package example

import "github.com/dstodev/silo"

func main() {
  if err := silo.Run("my-silo-id",
      silo.FromDockerImage("alpine:latest"),
      silo.WithArgs([]string{"echo", "Hello, world!"})); err != nil {
    panic(err)
  }
  silo.Wait("my-silo-id")
}
```

Calling on an already-running silo returns `silo.SiloRunning`. A stopped silo
may then run any new container. Subscribers persist and continue to receive
events throughout the container(s) lifecycle.

If the contained process exits on its own, the silo emits
`event.ContainerStopped` and transitions to stopped state automatically.

### `silo.Wait(id, options...)`

Wait for a silo's container to stop:

- If the silo exists but is already stopped, return immediately.
- If the silo does not exist, wait for the next container to start and then
  stop.

Accepts options:

- `silo.WaitSeconds()`: wait up to the specified duration, then return
  `silo.TimedOut` if the container is still running. Default is to wait
  indefinitely.

### `silo.Stop(id, options...)`

Stop a silo's container.

If the silo is already stopped or does not exist, return `silo.SiloNotRunning`.

Accepts options:

- `silo.StopSignal()`: use the specified signal instead of `SIGTERM` for the
  initial stop attempt.

- `silo.WaitSeconds()`: wait up to the specified duration for the process to
  stop before re-attempting more forcefully. Default is to wait indefinitely.

- `silo.TERM` (default): sends `SIGTERM` (or set stop signal) to the contained
  process, allowing it to exit gracefully. If the process does not exit within
  the timeout, it is forcefully killed with `SIGKILL`.

- `silo.CLOSE`: closes the silo's `stdin` stream, assuming the process will then
  exit gracefully. If the process does not exit within the timeout, the stop
  method falls back to `silo.TERM`.

- `silo.KILL`: sends `SIGKILL` to the contained process, stopping it
  immediately.

### `silo.Purge(id)`

Clear all state for a silo. A running silo does nothing and returns
`silo.SiloRunning`. A stopped silo will be purged, so:

- Subscribers are notified with `event.SiloPurged`
- Subscribers will not receive further events unless they subscribe again
- The container has stopped (subscribers first receive `event.ContainerStopped`
  where reason=`purge`)
- Daemon `silod` has forgotten event history for the silo
- The silo ID is available for reuse with `silo.Run()`.

If the silo does not exist, this method returns no error and has no effect.

### `silo.Subscribe(id, handler, options...)`

Subscribe to events from a silo.

The silo may be running, stopped, or non-existent at the time of subscription.

If the silo does not exist, for example, the daemon still waits for a container
to start with the silo ID, and the subscriber then receives events as they
occur.

After this call, the handler is called for each new event (either input or from
the silo) until the subscriber cancels the subscription. The handler is called
with a single argument of type `event.Event`. See the [Events
documentation](events.md) for event types and their additional values.

New subscribers optionally receive full event history up to the current moment,
then receive all new events as they occur. This allows subscribers to view a
silo's complete event history after it has started, stopped, started again, etc.
until the silo is purged or the subscriber disconnects.

Accepts options:

- `silo.FullHistory()`: receive all events from the silo's history before
  receiving new events. Incompatible with `silo.NoHistory()`.

- `silo.NoHistory()`: (default) only receive new events from the moment of
  subscription onward. Incompatible with `silo.FullHistory()`.

Supplying incompatible options returns `silo.InvalidArguments`.

```go
package example

import (
    "fmt"
    "log"

    "github.com/dstodev/silo"
    "github.com/dstodev/silo/event"
)

func myEventHandler(e event.Event) {
    switch ev := e.(type) {
    case event.ContainerStarted:
      fmt.Printf("Started %s\n", ev.SiloID)
    case event.ContainerStopped:
      fmt.Printf("Stopped %s (exit %d)\n", ev.SiloID, ev.ExitCode)
    case event.MessageFromContainer:
      fmt.Printf("[%s] %s\n", ev.StreamName, ev.MessageContent)
    case event.MessageFromProvider:
      fmt.Printf("#%d sent: %s\n", ev.ProviderPID, ev.MessageContent)
    case event.SiloPurged:
      fmt.Printf("Purged %s\n", ev.SiloID)
    }
}

func main() {
  sub, err := silo.Subscribe("my-silo-id", myEventHandler, silo.FullHistory())
  if err != nil {
    log.Fatal(err)
  }
  defer sub.Cancel()
  silo.Stop("my-silo-id")
}
```

### `silo.SendMessage(id, message)`

Send a message to the container `stdin` stream. The message is also delivered as
a `MessageFromProvider` event to all subscribers.

```go
package example

import "github.com/dstodev/silo"

func main() {
  if err := silo.SendMessage("my-silo-id",
      []byte("Hello, world!")); err != nil {
    panic(err)
  }
}
```

Sending input to a stopped or non-existent silo returns `silo.SiloNotRunning`,
and does not affect the silo's event history or state.

## Error Types

- `silo.SiloNotRunning`: the silo is stopped or does not exist
- `silo.SiloRunning`: the silo is already running
- `silo.TimedOut`: the wait time lapsed before the intended action completed
- `silo.InvalidArguments`: the provided arguments are invalid or incompatible
