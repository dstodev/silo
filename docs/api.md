# Silo API Reference

<!-- markdownlint-disable MD033 -->

[← README](../README.md)

This document describes the Silo API.

## Package `silo` functions

### `silo.Run(id, options...)`

Run a program in a silo.

Silo IDs must match the regular expression `[a-zA-Z0-9_.+-]+` and may not exceed
200 characters.

`silo.Run()` accepts options:

- `silo.FromDockerImage()`: Use a pre-built Docker image e.g. `alpine:latest`.
- `silo.FromDockerfile()`: (Re)build and use an image from a Dockerfile by path.
- `silo.WithArgs()`: Start the contained program with additional arguments.
- `silo.WithMount()`: Mount a directory from the host to the container. Call
  multiple times to mount multiple directories.

```go
package example

import "github.com/dstodev/silo"

func main() {
	s, err := silo.Run("my-silo-id",
		silo.FromDockerImage("alpine:latest"),
		silo.WithArgs([]string{"echo", "Hello, world!"}))
	if err != nil {
		panic(err)
	}
	s.Wait()
}
```

Calling `silo.Run()` on an already-running silo returns `SiloRunning`. A stopped
silo may be run again; if the contained process exits on its own, the silo emits
`EventStopped` and transitions to stopped state automatically.

### `silo.Stop(id, options...)`

Stop a silo's container. An already-stopped silo returns `SiloNotRunning`.
Accepts a stop method and a timeout parameter:

- `silo.TERM` (default): sends `SIGTERM` to the contained process, allowing it
  to exit gracefully. If the process does not exit within the timeout, it is
  forcefully killed with `SIGKILL`.

- `silo.CLOSE`: closes the silo's `stdin` stream, assuming the process will then
  exit gracefully. If the process does not exit within the timeout, the stop
  method falls back to `silo.TERM`.

- `silo.KILL`: sends `SIGKILL` to the contained process, stopping it
  immediately.

Calling `silo.New()` with an ID that already exists returns `SiloExists`.

Calling `silo.Purge()` on a running silo returns `SiloRunning`. Calling
`silo.Purge()` on a stopped silo will notify and unregister all subscribers,
then destroy all event history for the specified ID. The ID is freed and may be
reused with `silo.New()`.

### `silo.Subscribe(id, handler, options...)`

Subscribe to events from a silo.

New subscribers optionally receive full event history up to the current moment,
then receive all new events as they occur. This allows subscribers to view a
silo's complete event history after it has started, stopped, started again, etc.
until the silo is purged or the subscriber disconnects.

Accepts options:

- `silo.FullHistory()`: receive all events from the silo's history file, in
  order, before receiving new events.

- `silo.NoHistory()`: (default) only receive new events from the moment of
  subscription onward.

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

Sending input to a stopped silo returns `silo.SiloNotRunning`, and does not
affect the silo's event history or state.
