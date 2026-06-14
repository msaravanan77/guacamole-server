# guacamole-server — JIRA Issue Analysis

A record of a deep-dive triage of open Apache Guacamole JIRA issues, scoped to
**guacamole-server** (the C codebase: `libguac`, `guacd`, and the protocol
plugins). The goal was to surface genuine, code-level bugs of high community
value — memory-safety, concurrency, and build/compat — rather than clerical or
client-side items.

Each finding below was **verified against the source tree** (not just trusted
from the ticket text), with exact file/line references at the time of writing.

- Tracker: <https://issues.apache.org/jira/projects/GUACAMOLE/issues/>
- Branch where this analysis lives: `jira-analysis`

---

## Summary table

| Rank | Issue | Class | Verified present in tree | Severity |
|------|-------|-------|--------------------------|----------|
| 🥇 | [GUACAMOLE-2270](#guacamole-2270--abba-rwlock-deadlock-in-the-display-pipeline) | Concurrency hang | ✅ | High |
| 🥈 | [GUACAMOLE-2249](#guacamole-2249--out-of-bounds-write-in-the-nested-socket) | Memory corruption | ✅ | High |
| 🥉 | [GUACAMOLE-2273](#guacamole-2273--build-fails-against-freerdp--3250) | Compat / build break | ✅ | Medium (broad) |

Dropped after verification:
- **GUACAMOLE-2262** (CLIPRDR stack overflow) — **already fixed** in the current
  tree; the clipboard buffers are now heap-allocated via `guac_mem_alloc()`
  (PR #659 merged).
- **GUACAMOLE-1514** (OOB in `_freerdp_image_copy` on resolution mismatch) —
  real in spirit, but reported against 1.4.0; the display layer has been heavily
  rewritten since, so it needs fresh reproduction before it can be trusted.
- **GUACAMOLE-2121** (RDP microphone redirection broken in 1.6.0) — bisected by
  the reporter to **guacamole-client** (web app), not guacd; out of scope for
  this repo. See [Appendix](#appendix-guacamole-2121-not-a-server-bug).

---

## GUACAMOLE-2270 — ABBA rwlock deadlock in the display pipeline

**Severity:** High — whole-connection hang / black screen.
**Files:** `src/libguac/display.c`, `src/libguac/display-layer.c`, `src/libguac/display-flush.c`

### Mechanism — two lock orders that are mirror images

- **`guac_display_dup()`** (runs when a new user joins a connection) acquires:
  - `last_frame.lock` **(read)** — `src/libguac/display.c:242`
  - then, inside its per-layer loop, calls `guac_display_layer_get_bounds()`,
    which acquires `pending_frame.lock` **(read)** — `src/libguac/display-layer.c:51`
  - **Order: last_frame → pending_frame**

- **`guac_display_end_multiple_frames()`** (the render loop flushing a frame) acquires:
  - `pending_frame.lock` **(write)** — `src/libguac/display-flush.c:308`
  - then `last_frame.lock` **(write)** — `src/libguac/display-flush.c:323`
  - **Order: pending_frame → last_frame**

### The deadlock

When a user-join and a frame-flush interleave:

- **Thread A** (`guac_display_dup`) holds `last_frame` **read**, wants
  `pending_frame` **read** — but it is held **write** by Thread B → A blocks.
- **Thread B** (`guac_display_end_multiple_frames`) holds `pending_frame`
  **write**, wants `last_frame` **write** — but it is held **read** by Thread A → B blocks.

This is a textbook ABBA cycle, independent of reader/writer preference (A wants
to read something held for writing; B wants to write something held for reading).
glibc's tendency to queue new readers behind a waiting writer only widens the
window.

### Impact

The entire display pipeline stalls: protocol-client threads pile up on
`pending_frame.lock`, user-management paths block on the users-rwlock, and
**every viewer goes black for the remainder of the connection**. It is
intermittent and timing-dependent (the join must fire after `end_multiple_frames`
passes its early-return point), which makes it hard for users to report cleanly.

### Suggested fix shape

Impose a single global lock order. Cleanest option: make `guac_display_dup()`
stop reaching for `pending_frame` while holding `last_frame` — e.g. read each
layer's bounds from the **`last_frame` snapshot it already holds** instead of
calling `guac_display_layer_get_bounds()` (which reads `pending_frame`). `dup`
is resyncing *last_frame* state anyway, so pulling dimensions from
`pending_frame` is arguably incorrect as well as deadlock-prone. Alternatively,
reorder so both paths acquire in the same direction.

**Why it's the top pick:** it's a hang (worse than a crash for UX), it hits
multi-user / shared connections (a flagship feature), and it is under-reported
because of its intermittent nature.

---

## GUACAMOLE-2249 — Out-of-bounds write in the nested socket

**Severity:** High — heap / mutex corruption.
**File:** `src/libguac/socket-nest.c`

### Mechanism

The nested socket buffers up to `GUAC_SOCKET_NEST_BUFFER_SIZE` (7168) bytes. The
**write** path caps input at `sizeof(buffer) - written - 1`, reserving exactly
**one** spare byte for a temporary NUL terminator. The **flush** path
(`src/libguac/socket-nest.c:104-110`) scans the buffer to find the boundary of
the last *complete* UTF-8 character:

```c
while (length < data->written)
    length += guac_utf8_charsize(*(data->buffer + length));   // can overshoot

char overwritten = data->buffer[length];   // OOB read  when length > written
data->buffer[length] = '\0';               // OOB WRITE when length > written
```

`guac_utf8_charsize()` determines a character's width from the **lead byte
alone**. If the buffer's final byte is a multi-byte lead (e.g. `0xF0`, a 4-byte
UTF-8 lead) sitting at offset 7167, the loop adds 4 and `length` becomes
**7170 — three bytes past the end of the 7168-byte array**. The single reserved
guard byte is insufficient; the overshoot can be up to 3 bytes.

### Why it's real and nasty

`data->buffer` is a struct member immediately followed by other fields,
including a live `pthread_mutex_t buffer_lock`. The stray `'\0'` write corrupts
that mutex → undefined behavior, possible deadlock or crash. It is
**data-dependent** (only triggers when a partial multi-byte character lands
exactly at the buffer boundary), making it a maddening, rare field bug.

### Suggested fix shape

Clamp the scan so `length` never exceeds `data->written`: stop when the next
character would run past the real data and leave the partial trailing bytes for
the following flush. The existing `memcpy` shift (`src/libguac/socket-nest.c:123`)
already anticipates carrying a partial trailing character, so this matches the
original design intent. (Alternatively, size the guard region to the maximum
UTF-8 character width.)

**Caveat:** this nested-socket API is deprecated and lightly used, so the
exploitation surface is narrow — but it is genuine memory corruption in core
`libguac`.

---

## GUACAMOLE-2273 — Build fails against FreeRDP ≥ 3.25.0

**Severity:** Medium, but broad — blocks compilation for anyone on current FreeRDP.
**File:** `src/protocols/rdp/rdp.c:558`

```c
rdp_inst->Authenticate = rdp_freerdp_authenticate;
```

FreeRDP 3.25.0 deprecated the `Authenticate` callback in favor of
`AuthenticateEx`. Under FreeRDP's `WITHOUT_FREERDP_3x_DEPRECATED` guard, the
`Authenticate` field is compiled out, so this assignment **fails to compile**.
The current tree sets only `Authenticate` and has no `AuthenticateEx` path.

### Impact

Distributions and self-builders tracking current FreeRDP 3.x cannot build
guacd at all — a hard wall, not a degraded feature.

### Suggested fix shape

Detect the FreeRDP version (in `configure.ac` or via an `#if` on the FreeRDP
version macros) and assign `AuthenticateEx` — which has a different signature
(it adds a reason/flags parameter) — on new FreeRDP, falling back to
`Authenticate` on older versions.

**Note:** the ticket is marked *In Progress*; check for an existing PR before
duplicating the work.

---

## Appendix: GUACAMOLE-2121 (not a server bug)

**RDP microphone redirection does not work in guacamole 1.6.0.** Included here
only to document why it was excluded.

The reporter's own version matrix is a proof of where the regression lives:

| Combination | Result |
|-------------|--------|
| guacamole-client 1.5.5 + guacd 1.5.3 | ✅ works |
| guacamole-client 1.5.5 + guacd 1.6.0 | ✅ works |
| guacamole-client 1.6.0 + guacd 1.6.0 | ❌ fails (silent mic) |

Because the **same guacd 1.6.0** works when driven by the 1.5.5 client, the
guacd audio-input path (`src/protocols/rdp/channels/audio-input/`) is
demonstrably functioning; the only changed variable in the failing case is the
web client. Code inspection agrees: guacd either rejects an unsupported mimetype
outright or faithfully forwards the PCM it receives — it cannot turn non-silent
input into silence. The fix belongs in **guacamole-client**
(`Guacamole.RawAudioRecorder`), not in this repository.

---

*Generated as part of a triage session. File/line references reflect the state
of the tree at analysis time and should be re-confirmed before acting.*
