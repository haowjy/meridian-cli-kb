# Harness Connection Seams

## Connection ownership

`SpawnManager` consumes the connection event stream and fans events out. Claude's stream permits one consumer, so another reader would compete with the drain loop. See [attempt facts and delivery](../architecture/attempt-facts-and-delivery.md#the-emit-path).

## Runtime approval policy

A request handler separates who decides approval requests from how the transport sends them. See the connection protocol in `src/meridian/lib/harness/connections/`.

## Managed-primary passthrough

The user's TUI sends input directly to the harness backend; routing that input through Meridian would add latency and complexity. Claude primaries use a PTY because Claude has no separate attachable backend.

## Nesting

The spawn depth cap bounds recursion across commands. A per-command launch guard also blocked delegated agents from testing launches.
