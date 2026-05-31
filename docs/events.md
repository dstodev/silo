# Events

<!-- markdownlint-disable MD033 -->

[← README](../README.md)

Describes Silo events from the event subscriber's perspective.

---

## All events

All events contain values:

- Silo ID
- Event ID
- Time of event creation or receipt by daemon `silod`

Events are timestamped, assigned a unique ID, then recorded as received by
daemon `silod`. Event IDs are monotonically increasing integers starting at 1.
Each silo maintains its own independent sequence.

---

## Control events

All control events contain additional values:

- Provider PID

### `event.ContainerStarted`

The contained program has started.

This event contains additional values:

- Container ID
- Full program launch command (container entrypoint + command + arguments)
- Environment

### `event.ContainerStopped`

The contained program has stopped.

This event contains additional values:

- Exit code
- Signal (if applicable)
- Reason (`stop`, `purge`, or `nil` if
  the container stopped on its own)

If the container stopped on its own, provider PID is `nil` and reason is `nil`.
If the container was stopped by a call to `silo.Stop()`, provider PID is the
caller's PID and reason is `stop`. If the container was stopped by a call to
`silo.Purge()`, provider PID is the caller's PID and reason is `purge`.

### `event.SiloPurged`

The silo has been purged, so:

- The container has stopped (you will first receive `event.ContainerStopped`
  where reason=`purge`)
- Daemon `silod` has forgotten event history for the silo
- The silo ID is available for a new container to start within.
- You (the function subscribed to silo events) will not receive further events
  for the silo (unless you subscribe again)

---

## Message events

All message events contain additional values:

- Stream (`stdin`, `stdout`, or `stderr`)
- Message content

### `event.MessageFromContainer`

The contained program printed a message via `stdout` or `stderr`.

### `event.MessageFromProvider`

The contained program received a message via `stdin`.

This event contains additional values:

- Provider PID

---

## Notes

A process subscribed to a silo that sends a message to `stdin` will receive a
message event listing its own PID as the provider.

Event history is stored in a JSON-formatted file per silo within the daemon's
workspace, and retained until calling `silo.Purge()`.
