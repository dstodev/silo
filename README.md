# Silo

<!-- allow HTML in Markdown for explicit formatting e.g., line breaks: -->
<!-- markdownlint-disable MD033 -->

Silo is a Golang library for running programs in ephemeral but logically
persistent containers. A silo holds up to one container at any time: a container
may start, stop, restart, or be replaced, but subscribers observe a single
continuous event stream for the lifetime of the silo.

All behaviors are independent of the calling process:

- "Process A" may start a program in a silo
- "Process B" may subscribe to events from the silo
- "Process C" may send events to the silo
- "Process D" may stop the program running in a silo

Processes A, B, C, and D may be the same or different.

## Documentation

| Document                     | Description                                         |
| ---------------------------- | --------------------------------------------------- |
| [Events](docs/events.md)     | Event types, history, lifecycle, and stop methods   |
| [Daemon](docs/daemon.md)     | `silod` architecture, socket, workspace, goroutines |
| [API Reference](docs/api.md) | Full public API                                     |

## References

- Written in `Go`
  - [Homepage](https://go.dev)
- Docker for image & high-level container management
  - [Docker](https://www.docker.com)
- `containerd` — Docker's underlying container runtime; its types may be
  referenced directly in the implementation
  - [Homepage](https://containerd.io)
  - [GitHub](https://github.com/containerd/containerd)
