# Relay protocol — captured ground truth

Date: 2026-10-31. Captured by running the real `bavarian-ssh-client.exe` against a
local fake car and a local fake server. **Nothing was sent to `pretzel.bavarian-aa.uk`.**

## Correction: the server is not TurboCode

The exe contains **zero** references to `turbocode` or `zerotier` — only
`pretzel.bavarian-aa.uk`. The TurboCode / ZeroTier references live solely in the
repo's `install.md`, which documents a different, paid route. The `bavarian-ssh-client`
route is independent and free. My earlier attribution was wrong.

## Confirmed: the VIN is sent to the server

You were right. The client sends a three-field identity message. Exact capture below.

## Client to server — confirmed by capture

Framing: `[tag:u8][u32be len][body]`, repeated per field.

Observed, byte for byte, reproducible across 6 runs:

```
06 00000005 30 2e 31 2e 30                       tag 0x06  version   "0.1.0"
01 00000000 00000000 00000000                    tag 0x01  identity  3 empty fields
02 0000000a 73 73 68 5f 65 6e 61 62 6c 65        tag 0x02  op        "ssh_enable"
```

Sent in that order, immediately on connect, ~1 ms after the car TCP socket opens.

**Tag 0x01 is the identity message: three length-prefixed strings.** They are empty
here only because the car IP was typed in manually, so discovery never ran and the
client had no VIN/MAC/DIAGADR to put in them. This matches the static analysis exactly:
three owned `String`s move out of the task state (slots `+0x78`, `+0x90`, `+0xa8`,
stride 0x18) into the hello struct alongside `"0.1.0"` and `"ssh_enable"`.

**Field order within tag 0x01 is still unpinned** — the three were empty in this
capture. Discovery parses the markers in the order `DIAGADR`, `BMWMAC`, `BMWVIN`, and
the UI prints VIN, diag address, MAC. Needs a populated capture to settle.

Worth noting for the app: the client tolerates sending an **empty** identity. So a
manually entered IP is a viable path, if the server accepts it.

## Car discovery — SOLVED (captured)

The client **UDP-broadcasts an HSFZ identification request**:

```
destination   UDP port 6811   (broadcast)
source        UDP port 7811
payload       00 00 00 00 00 11
              = HSFZ frame, length 0, control word 0x0011 (ident request)
```

Captured directly off the wire from the real client at `192.168.1.204:7811`.

Note: the port is **6811**, not 68001. (68001 isn't a valid port number — that was my
error earlier. It's also why no `68001` constant turned up in the binary.)

The car answers with control word **0x0012** and a payload containing the three
markers `DIAGADR`, `BMWMAC`, `BMWVIN`.

### Ident reply payload format — unknown, six layouts ruled out

Tested against the real client, all replying with ctrl `0x0012` unicast back to the
client's source port 7811. **Every one was rejected** ("No vehicles found"):

| Layout | Result |
|---|---|
| `DIAGADR=10\nBMWMAC=...\nBMWVIN=...\n` | rejected |
| `DIAGADR=10;BMWMAC=...;BMWVIN=...;` | rejected |
| `DIAGADR=10\0BMWMAC=...\0BMWVIN=...\0` | rejected |
| `DIAGADR10BMWMACAABBCCDDEEFFBMWVIN...` (fixed-width, no separator) | rejected |
| `<DIAGADR>10</DIAGADR><BMWMAC>...</BMWMAC><BMWVIN>...</BMWVIN>` | rejected |
| `BMWVIN...BMWMAC...DIAGADR10` (VIN first) | rejected |

So the answer to "is it `key=VINNUMBER`?" is **no** — at least not with `=` and any of
the separators above.

The client also does **no** link-local filtering: there is no `0xa9fe` constant and no
169/254 comparison anywhere in `.text`, so replies are not rejected for coming from
outside `169.254.0.0/16`. (The 169.254 expectation is operational knowledge about the
ENET adapter, not something this client enforces. Our own warning about addresses
outside that range stays a deliberate addition.)

Remaining hypotheses, untested: a binary key/length/value encoding; a required extra
field; a different control word on the reply; or the ident happening over TCP 6801
after the broadcat only locates the address.

**Stopping here deliberately.** Six wrong guesses is enough evidence that this is not
worth brute-forcing; the car's own reply settles it in one observation.

### Superseded: original note

A `KEY=value\n` text payload was **rejected** ("No vehicles found"), so that is not the
layout. The parser does no delimiter comparisons at all — the only byte constants in it
are `0xc0` / `0xbf` / `0x80`, which are UTF-8 char-boundary checks for string slicing.
That points to **fixed-width values immediately following each marker, with no
separator**, but I did not get a clean test of that hypothesis (the patch that was
supposed to switch formats silently failed to apply, so only the `KEY=value` form was
ever actually sent).

This is best settled by one real capture rather than more guessing: the car's own reply
is authoritative, and any guess risks baking in a parser that works on a fake and fails
on the vehicle.

### Consequence for the Android app

Discovery is a UDP broadcast, so the app must:

- bind its UDP socket to the **Ethernet** interface (`bindProcessToNetwork`), not Wi-Fi;
- enable `SO_BROADCAST`;
- send from source port **7811** to destination port **6811**;
- accept the reply and parse the three markers out of it.

Binding to the right interface matters more here than for a TCP connect: a broadcast on
the wrong interface silently goes nowhere.

## Car to server forwarding — SOLVED (captured)

Captured by having a fake car push a frame at the client over the proven manual-IP
path. The client wrapped it and sent:

```
03 0000000e  00 00 00 08 00 01 48 45 4c 4c 4f 43 41 52
tag 0x03     <-------- exactly the bytes the car sent -------->
```

**Tag `0x03` = car data, client to server.** Two things follow, and the second is the
important one:

1. The body is the **raw bytes read off the car socket, passed through verbatim** —
   HSFZ header included. The client does not re-frame, re-encode, or even parse them.
2. Therefore **the relay is a byte pipe, not an HSFZ-aware proxy.** The HSFZ codec is
   used for *discovery only*. For the relay, the Android port can read the TCP socket
   and forward whatever it gets, without understanding any of it.

That removes a whole class of risk from the port: there is no framing logic to get
subtly wrong in the forwarding path.

## Server to client — SOLVED (static + confirmed by experiment)

Found in the dispatch at `0x14001d572`:

```
movzx ecx, al            ; al = the tag byte
cmp   ecx, 0x81  je ...  ; -> bytes reader  (0x14005dd40)
cmp   ecx, 0x82  je ...  ; -> string reader (0x14005dcba)
cmp   ecx, 0x83  jne err ; -> string reader (0x14005dcba)
                         ; anything else: "unknown server tag"
```

**Server to client tags are `0x81`, `0x82`, `0x83`. Nothing else is valid.**

| Tag | Reader | Meaning |
|---|---|---|
| `0x81` | `Bytes` | **raw bytes to write to the car socket** |
| `0x82` | `String` | text for the user (log line; a "log kind" byte sub-types it) |
| `0x83` | `String` | text for the user (status / completion) |

Confirmed experimentally. Sending `81 0000000b 00000005 0001 "PROBE"` caused the client
to write `00 00 00 05 00 01 50 52 4f 42 45` **verbatim** to the car socket; the fake car
replied, and the client forwarded that reply back up as tag `0x03`:

```
03 0000003e  00 00 00 38 00 12 44 49 41 47 41 44 52 3d 31 30 ...
```

Tags `0x82` and `0x83` sent nothing to the car, as predicted.

### Why the earlier scan missed this

Rust stores these error strings **length-prefixed**, and the code `lea`s the *length
byte*, one byte before the text. Searching for references to the text address found
nothing and I wrongly concluded the strings were unreferenced. Searching a small
*range* found all of them immediately. The earlier sweep of tags `0x00`–`0x04` was also
looking in the wrong place entirely — the server tags were never low numbers.

### The pipe is fully specified

```
server --[0x81][u32be len][raw]-->  client  --raw bytes-->      car
car    --raw bytes-------------->  client  --[0x03][u32be len][raw]--> server
server --[0x82|0x83][u32be len][utf8]--> client --> show to user
```

Both directions are **verbatim byte passthrough**. The relay needs no HSFZ awareness at
all; HSFZ framing is used only by discovery.

## Superseded: earlier partial bounding

Same framing. The client reports bad tags as:

```
Bad message from the server: malformed relay message: unknown server tag 0x04
```

- **0x04 — confirmed invalid.**
- **0x00–0x03** did not raise that error in the probe window; screenshots
  `shot_00..03.png` in the scratchpad will confirm individually. Valid server tags are
  almost certainly a subset of `{0x00, 0x01, 0x02, 0x03}`.
- None of 0x00–0x04 caused the client to forward anything to the car, so the
  "car data" tag was not identified. The probe body was a well-formed HSFZ frame
  (`ctrl 0x0001`, payload `PROBE`); the fake car logged nothing in every case.

Likely reason: the client waits for some server message to open the session before it
starts piping, so a bare data message out of sequence is ignored. Resolving this needs
either a populated/real session capture or more static tracing.

## Also confirmed by this run

- **Car port is 6801**, shown in the UI: `Connecting to the car at 127.0.0.1:6801...`.
- **Order of operations: car first, then server.** `Connecting to the car at ...` →
  `Connecting to the server...` → `Connected.` With no car reachable, the client never
  dials the server at all. This is why a no-car capture needs a fake car.
- **Discovery does not probe 127.0.0.1.** The client auto-searches on startup; the fake
  car on loopback was never contacted. So discovery enumerates real interfaces — it does
  not sweep loopback. Consistent with a link-local sweep or broadcast.
- The HSFZ framing recovered from the disassembly was **accepted by the real client**
  (it connected and proceeded), validating the codec:
  `[u32be payload len][u16be ctrl][payload]`.

## Harness (scratchpad, reusable)

- `fakecar.py` — HSFZ listener on 127.0.0.1:6801, logs frames, answers ident.
- `fakeserver.py` — logging-only WebSocket server, decodes the framing.
- `probeserver.py` — sends one candidate server tag per connection.
- `sweep.py` — restarts the client, drives the GUI, runs one tag per cycle.

To continue the tag hunt: `python sweep.py 5 6 7 8` and read `shot_*.png`.

---

## Corrections from a real car (2026-10-31)

A live run against the J29 settled three things that static analysis and the
loopback captures had got wrong.

### The server→client tags are NOT uniformly framed

Captured bytes:

```
82 03 00 00 00 0a  "Gate: open"
82 02 00 00 00 0f  "Enabling SSH..."
82 03 00 00 00 0c  "SSH: enabled"
81 00 00 00 0c     00 00 00 06 00 01 f4 63 31 01 fd ac
83 01
```

Decoded:

| Tag | Layout |
|---|---|
| `0x81` | `[tag][u32 len][raw bytes]` — straight to the car |
| `0x82` | `[tag][u8 kind][u32 len][utf8]` — **has a kind byte** (2 = step, 3 = result) |
| `0x83` | `[tag][u8 code]` — **no length at all**; `0x01` = success |

The kind byte on `0x82` is the "log kind" the desktop client's error strings
mention. Parsing `0x82` without it reads the length four bytes early and
produces values like 50331648 — which is exactly how the bug surfaced.

`0x83 0x01` arrives immediately after "SSH: enabled" and the server then closes,
so it is the authoritative end-of-job signal. A socket close after it is normal,
not a failure.

### The relay works

The whole pipe ran correctly on the car: discovery found it, the hello went out,
`0x81` payloads were written to the car verbatim, car replies came back as
`0x03`, and the server reported **SSH: enabled**. Only the parsing of the two
text/status tags was wrong, so the app failed to notice it had succeeded.

### The SSH host is a different machine from the diagnostic host

The ENET gateway answered HSFZ on 6801 at `172.16.153.57` but refused port 22
with **ECONNREFUSED** — reachable, nothing listening. The head unit is a
separate machine, which is why install.md SSHes to a different address
(`169.254.199.99`) than the diagnostic tooling uses.

So the app no longer assumes one address serves both. After SSH is enabled it
looks for whoever is listening on port 22: the documented head-unit addresses
first, then DHCP/gateway candidates, then a sweep of the local /24. The Install
step shows both addresses separately and lets the head unit IP be set by hand.

### Addressing is not link-local on this car

The ENET link handed out `172.16.153.0/24` over DHCP, not `169.254.0.0/16`. The
warning about addresses outside link-local was wrong and alarming; the check now
accepts link-local, `172.16/12`, `192.168/16`, `10/8` and BMW's `160.48/16`.

An earlier version also *invented* a candidate address by taking the phone's
subnet and appending `.99`, copying the `169.254.199.99` convention. On this car
that produced `172.16.153.99`, which nothing answers at, and offered it ahead of
the real gateway. Guessing is gone; the sweep finds the real host instead.

---

## Head unit layout, confirmed on the car (2026-10-31)

Established by running the app against the J29. These are facts now, not guesses.

| | |
|---|---|
| Diagnostics (HSFZ 6801) | `172.16.153.57` — the ENET gateway |
| SSH (port 22) | `172.16.153.204` — found by sweeping the /24 |
| Phone | `172.16.153.182` |
| QNX nodes visible from the SSH host | `hu-jacinto`, `hu-omap` |
| HMI tree | `/net/hu-omap/fs/sda0/opt/hmi/` → contains `ID5`, `otf`, `scripts` |

So the real install paths are:

```
root     /net/hu-omap/fs/sda0
remount  mount -o remount rw /net/hu-omap/fs/sda0
daemon   /net/hu-omap/fs/sda0/opt/aa/nbt-aad
plugin   /net/hu-omap/fs/sda0/opt/hmi/ID5/data/ro/Plugins/NbtAad
```

which is exactly what install.md describes, reached without the `on -f` shell hop.

**ID5 on an ID6 unit is confirmed.** The directory really is called `ID5` on
MB-007.007.008 NBT EVO iDrive 6. There is no ID6 directory.

### Three bugs this exposed

1. **The SSH host is not the diagnostic host.** `.57` answers HSFZ on 6801 but
   refuses port 22 with `ECONNREFUSED` — reachable, nothing listening. Assuming
   one address served both was wrong.
2. **The writable tree is under `fs/sda0`, not `/opt`.** `/net/hu-omap/opt/...`
   does not resolve. Only `/net/hu-omap/fs/sda0/opt/...` does.
3. **This unit's `ls` lists `.` and `..`.** Unfiltered, the first HMI candidate
   became `.` and the probe asked about `/opt/hmi/./data/ro`.

### Also learned

- `touch` exits **126** (exec failure) on this unit, so a write test built on it
  reports "not writable" regardless of permissions. Shell redirection works.
- The ENET link is `172.16.153.0/24` via DHCP, not `169.254.0.0/16`.
