# Silo API Reference

<!-- markdownlint-disable MD033 -->

[← README](../README.md)

This document describes the Silo API.

## Functions

### `silo.Run(id, options...)`

Run a program in a silo.

Silo IDs:

- Match regular expression: `[a-zA-Z0-9_.+-]+`
- May not exceed 20 characters

`silo.Run()` accepts options:

- `silo.FromDockerImage()`: Use a pre-built Docker image e.g. `alpine:latest`.
  Incompatible with `silo.FromDockerfile()`.
- `silo.FromDockerfile()`: (Re)build and use a Dockerfile by path.
  Incompatible with `silo.FromDockerImage()`.
- `silo.WithArgs()`: Start the container with additional arguments.
- `silo.WithMount()`: Mount a directory from the host to the container. Supply
  multiple times to mount multiple directories.

Exactly one container image source is required.

Supplying invalid ID or options returns `silo.InvalidArguments`.

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

Calling `silo.Run()` on an already-running silo returns `silo.SiloRunning`.

After a silo stops, it may then `Run()` again.

### `silo.Wait(id, options...)`

Wait for the program to stop:

- If the silo with ID exists but is already stopped, return immediately.
- If the silo with ID does not exist, wait for silo with ID to start and stop.

Accepts options:

- `silo.WaitSeconds()`: wait up to the specified duration, then return
  `silo.TimedOut` if the container is still running. Default is to wait
  indefinitely.

### `silo.Stop(id, options...)`

Stop the program.

If the silo with ID is not running or does not exist, returns
`silo.SiloNotRunning`.

Accepts options:

- `silo.TERM` (default): sends `SIGTERM` (or signal set by `silo.StopSignal()`)
  to the contained process, allowing it to exit gracefully. Waits up to 30
  seconds (configurable by `silo.WaitSeconds()`) for the process to stop, then
  forcefully terminates the process with `SIGKILL` if it is still running.

- `silo.CLOSE`: closes the silo's `stdin` stream, assuming the process will then
  exit gracefully. Waits up to 30 seconds (configurable by `silo.WaitSeconds()`)
  for the process to stop, then falls back to `silo.TERM` behavior with the same
  timeout duration, meaning it will wait twice the specified duration before
  forcefully terminating with `SIGKILL`.

- `silo.KILL`: sends `SIGKILL` to the contained process, stopping it
  immediately.

- `silo.StopSignal(s)`: use signal `s` instead of `SIGTERM` to signal the
  process to stop.

- `silo.WaitSeconds()`: wait up to the specified duration for the process to
  stop before re-attempting more forcefully. Default is 30 seconds. 0 to wait
  indefinitely for the process to exit.

### `silo.Purge(id)`

Clear all state for a stopped silo with the given ID:

- Subscribers are notified with `event.SiloPurged`
- Subscribers will not receive further events unless they subscribe again
- The container has stopped (subscribers first receive `event.ContainerStopped`
  where reason=`purge`)
- Daemon `silod` has forgotten event history for the silo
- The silo ID is available for reuse with `silo.Run()`.

If the silo with ID is running, returns `silo.SiloRunning`.

If the silo with ID is not running or does not exist, returns
`silo.SiloNotRunning`.

### `silo.Subscribe(id, handler, options...)`

Subscribe to events from a silo.

The silo may be running, stopped, or non-existent at the time of subscription.

If the silo does not exist, the daemon waits for one to start with the given ID,
then supplies events to subscribers as they occur.

After this call, the handler is called for each new event until the subscriber
cancels the subscription. The handler is called with a single argument of type
`event.Event`. See the [Events documentation](events.md) for event types and
their additional values.

New subscribers optionally receive full event history up to the latest event,
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
