# CypherCon 9 Badge Firmware Patches

This repository contains patched firmware for the CypherCon 9 badge (tymkrs, RP2040/MicroPython). Two vulnerabilities were identified and fixed that could be triggered by crafted IR packets from other badges.

## Vulnerability 1: Firmware Crash via Command/Chat Packet (c5abb13)

**Severity:** High — remote denial of service

The `handle_command()` and `handle_chat()` functions expected arguments, but their callsites in `handle()` (`main.py:783-785`) invoked them with no arguments. Sending a crafted IR packet with `event_id` 5 (command) or 7 (chat) would trigger a `TypeError` exception, causing the MicroPython main loop to exit and the badge to become unresponsive.

**Fix:** Updated the function signatures to take no arguments, matching the callsites. Both are currently stubs (`pass`) for unimplemented features:

```python
def handle_command():
    pass

def handle_chat():
    pass
```

## Vulnerability 2: Badge Crash via Alias Memory Buffer Overwrite (6eb0a01)

**Severity:** High — remote denial of service

The `write_alias_memory()` function writes 16 bytes to the `alias_memory` file at offset `remote_index * 16`. The `remote_index` value comes directly from the `from_index` field of a received IR packet (a 2-byte field, max 65535), which flows through `handle_page()` / `handle_broadcast()` -> `note_alias()` -> `write_alias_memory()` with no bounds checking.

A crafted packet with a large `from_index` causes a write far beyond the intended file boundary, crashing the badge. Initial testing suggests this does not permanently brick the device.

**Fix:** Added a bounds check in `write_alias_memory()` (`main.py:1263`) that silently drops writes with an out-of-range index:

```python
def write_alias_memory(remote_index, alias_bytearray):
    if (remote_index <= 700):
        # ... proceed with write
```

The limit of 700 matches the valid badge serial number range (the `next_alias` function at line 1276 wraps at index 675).
