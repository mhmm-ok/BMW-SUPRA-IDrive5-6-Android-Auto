# Step 1 findings — nbt-aad verification, relay analysis, blockers

Date: 2026-10-31

## 1. nbt-aad is genuine, and matches this source tree exactly

**Verdict: verified. Pin it.**

```
SHA-256  85adeab90783a993ccf404ab82cf9d227d6b859d57142cc4c5f2790fa320aa33
size     340,117 bytes
```

Evidence:

| Check | Result |
|---|---|
| ELF type | `ET_EXEC`, ARM EABI5, 32-bit LSB — a true QNX executable, not ET_DYN. The Makefile comments specifically call this out ("Remove -fPIC and -Bdynamic to produce a true QNX executable (ET_EXEC)"). |
| Interpreter / OS | `/usr/lib/ldqnx.so.2` plus a `QNX` ELF note. Genuine QNX Neutrino ARM binary. |
| `DT_NEEDED` | `liba2i_client.so`, `libsocket.so.3`, `libssl.so.2`, `libcrypto.so.2`, `libscreen.so.1`, `libcpp.so.4`, `libc.so.3` — exactly the Makefile's `-la2i_client -lsocket -lssl -lcrypto -lscreen` plus `-lang-c++` (Dinkum `libcpp`, i.e. the `_cpp` qcc variant, not `_gpp`). |
| `DT_RPATH` | `/opt/mm/lib:/lib:/usr/lib` — character-for-character the Makefile's `-Wl,-rpath,...`. |
| Compiler | `GCC 4.4.2` x21, which is the QNX SDP 6.5.0 toolchain `build.sh` documents. |
| Translation units (not stripped) | The 18 `STT_FILE` entries are **exactly** the Makefile's `SRCS` list, in the same order: main, audio_manager, a2i_stream, audio_route, pcm_utils, video_manager, tcp_server, aa_session, aa_cryptor, bluetooth_manager, wireless_setup, touch_probe, dsi_client, voice_scenario, bt_play, ccs_input, sdr_builder, crash_handler. |
| Version string | `Build: v1.8.49-sppdir`, built `Oct 30 2026 20:59:23`. Matches `#define NBT_AAD_BUILD "v1.8.49-sppdir"` in `src/main.cpp:1328`. (The `readme.md` header still says v1.8.47-gesture3 — stale doc, not a mismatch.) |
| Env vars | All 113 `NBT_*` names in the binary exist in the source. Zero binary-only names. |
| Log strings | 577 candidate log messages checked: 559 matched the source verbatim. All 18 "misses" were string literals split across source lines (each verified individually) or libcpp internals such as `vector<T> too long`. |
| License notice | `Required Notice: Copyright (c) 2026 nanaisu - NBT-AAD`, PolyForm Noncommercial 1.0.0 — intact. |

Conclusion: this is the project's own daemon, built one day ago from the source in `githubrepo/`. Safe to bundle and pin.

**Licensing note:** PolyForm Noncommercial 1.0.0. Bundling is fine for your own car; selling the app, or bundling it with anything sold, is not permitted, and the Required Notice must ship with it.

---

## 2. bavarian-ssh-client.exe — relay recovered, nothing uncopyable

A Rust `iced` GUI app (`tokio` + `tokio-tungstenite` + `rustls`) with three internal crates:
`bavarian-hsfz` (`discovery.rs`, `hsfz.rs`), `bavarian-relay` (`lib.rs`), and `bavarian-ssh-client` (the binary).

**Answer to your stop-condition: there is no runtime token, no key, no credential and no device identifier in the hello.** Everything the client sends is reproducible from the binary.

### Transport to the car: HSFZ over TCP 6801

`crates/bavarian-hsfz/src/hsfz.rs`. I recovered both directions of the codec from the disassembly — the encoder at `0x14005d5fa` and the decoder at `0x14005d6ca`.

Frame layout:

```
offset 0..4   payload length, u32 big-endian   (counts payload ONLY, header excluded)
offset 4..6   control word,  u16 big-endian
offset 6..    payload (length bytes)
```

The decoder requires `buf.len() >= 6`, then `buf.len() >= 6 + length`, then drains exactly `6 + length` bytes and keeps the remainder buffered. So the Kotlin port must treat the TCP connection as a byte stream with a re-entrant framer, not as datagrams.

The control word is a **parameter** to the encoder, never a constant. The client does not decide it — it forwards whatever the server asks for. This is the structural reason the tool is a dumb pipe.

Car port: **TCP 6801** (`mov word ptr [...], 0x1a91`, two sites). The standard HSFZ diagnostic port.

### Car discovery: identification reply parsed as ASCII markers

`crates/bavarian-hsfz/src/discovery.rs`. The reply is scanned for three literal markers, stored as a 3-entry table at `0x1404847B0`:

```
"DIAGADR"   "BMWMAC"   "BMWVIN"
```

rendered in the UI as `  VIN `, `  diag address `, `  MAC `. So the VIN for your confirm screen comes straight out of the identification reply — no extra UDS read required.

`ws2_32.dll` imports include `sendto`/`recvfrom` alongside `connect`/`send`/`recv`, and the discovery task carries an `at least one target` string, so it sweeps candidate targets rather than relying on one hardcoded address. I did **not** find a `68001` constant, so I have not yet pinned whether the probe is a UDP broadcast or a TCP sweep on 6801. I'll settle that during the port rather than guess, since it decides how the Android side binds and addresses its socket.

### Server side

Default `wss://pretzel.bavarian-aa.uk:8443`, overridable. Full CLI surface:

```
Usage: bavarian-ssh-client [--server URL] [--insecure]
  --server    websocket server URL, ws:// or wss://
  --insecure  accept any TLS certificate from the server
              (always on in debug builds)
```

A plain `tokio-tungstenite` client handshake, with no custom headers beyond what the library itself sends.

### The hello

Two literals, both `String`-copied into the handshake struct at `0x14001fac1` and `0x14001fb3c`:

```
version = "0.1.0"
op      = "ssh_enable"
```

That is the whole hello.

### Relay framing

`bavarian-relay/src/lib.rs`, length-prefixed binary rather than JSON. From the reader at `0x14005dd40`:

```
u32 big-endian length, then that many body bytes
```

with error paths `missing length`, `body shorter than its N-byte length`, `message ended early`, `trailing bytes`. Messages are tagged (`unknown server tag N`, `unknown log kind N`), and the server can push log lines to the client for display — which maps straight onto your monitor view.

I have the length framing and the hello exactly. The **tag numbering** is inlined into a 6,500-instruction async function and I have not finished enumerating it. That is the remaining work for step 2; it is recoverable by tracing, and I won't guess at it.

### Your SSH-enable guard

`31 01 10 0E` does **not** appear in the client binary. Neither does `31 01 FD EE 38`. Confirmed by byte search over the whole file. This matches your own description of the tool: the server supplies the commands.

This has a real design consequence. The app **cannot** "enable SSH only via the known-good routine" by choosing to send `31 01 10 0E`, because it never composes that command — it only forwards bytes. So the guard has to be the second thing you described: a **filter on the relayed stream**. Decode every HSFZ frame in both directions, and if a payload ever begins `31 01 FD EE 38`, drop the frame, abort the session, and surface it loudly. That is implementable and I will build it that way.

Be clear-eyed about what that buys: the enable step's correctness rests on a third-party server (TurboCode's) behaving, with our filter as a backstop against the one known-destructive routine. It is not us issuing a vetted command.

---

## 3. ~~Blocker~~ RESOLVED: release.zip supplied and verified

`release.zip` arrived 2026-10-31 and all five install-set files check out. **The blocker below is cleared**; the full install set is staged in `verified/` with `SHA256SUMS.txt`.

### libNbtAad.so — verified

```
SHA-256  66d171d8c4f931191bb9cf85d2320b153bdc914036fbdb6841c92fa575ef7ded
size     49,404 bytes
version  rhmiapp-20, built Oct 29 2026 14:51:41
```

| Check | Result |
|---|---|
| ELF type | `ET_DYN` ARM EABI5 shared object — correct, built `-shared -fPIC` per the Makefile's `PLUGIN_CXXFLAGS`. |
| `DT_SONAME` | `libNbtAad.so` — exactly the Makefile's `-Wl,-soname,libNbtAad.so`. |
| `DT_NEEDED` | `libAppFramework.A.so`, `libCoreFramework.A.so`, `libstdc++.so.6`, `libm.so.2`, `libc.so.3` — the first two are precisely the Makefile's `$(RHMI_LIBS)`, and `libstdc++.so.6` confirms the `_gpp` qcc variant the Makefile specifies ("GNU libstdc++ to match BMW's plugins"), distinct from the daemon's `libcpp.so.4`. |
| Translation units | One: `nbt_aad_app.cpp`. Exactly the Makefile's `rhmiapp` rule. |
| Compiler | `GCC 4.4.2` — same SDP 6.5.0 toolchain as the daemon. |
| Log strings | 74 candidates checked, 73 matched `rhmiapp/nbt_aad_app.cpp` verbatim. The single miss is the version string itself. |
| Deploy targets referenced | `/opt/aa/nbt-aad`, `/opt/aa/nbt-aad.env`, the `*.disable` flags, `/var/opt/aa/{autostart,bootguard,menu-idrive,usblog}` — confirms the daemon path the app must write. |

### Version note: rhmiapp-20, not -22

The released `.so` is **rhmiapp-20**; the source tree is rhmiapp-22. This is not a defect and not a mismatch to fix — rhmiapp-20 is what the project currently publishes, and we cannot build -22 (no QNX licence). Shipping the released artifact is the right call.

The release set is **internally self-consistent**, which is what actually matters for HmiMain3D to load it:

- `NbtAad.manifest` — byte-identical to the source tree.
- `NbtAad_ALL_Images.zip` — byte-identical to the source tree.
- `NbtAad.xml` references textIds `1, 10, 11, 12, 15, 16, 18, 19`; every one is present in the release `NbtAad_ALL_Texts.zip` (10 locales). No dangling references.
- The source tree's XML additionally references textId `20`, which the release Texts zip does not contain — that is the rhmiapp-22 "Show driving mode changes" checkbox. The release XML correctly doesn't reference it.

So the only functional difference from the source tree is the absence of that one checkbox, which is off by default anyway. Everything else is there: Start Android Auto, Force Stop, Restart, Start automatically, Save logs to USB, MENU key opens iDrive, About.

Pairing with the daemon is also correct: `install.md` has you download "the latest nbt-aad and release.zip" as **separate** assets, so `nbt-aad` v1.8.49 (Oct 30) alongside the rhmiapp-20 release set is exactly the combination the project's Releases page currently offers.

### Final pinned install set (`verified/SHA256SUMS.txt`)

```
85adeab90783a993ccf404ab82cf9d227d6b859d57142cc4c5f2790fa320aa33  nbt-aad
66d171d8c4f931191bb9cf85d2320b153bdc914036fbdb6841c92fa575ef7ded  libNbtAad.so
cc09123717fea5c41698c1ff4c08b4cd5dde620fda1b3d00b0e8699a3c621af2  NbtAad.manifest
c02dfc01591bf8416496e687819c4ec7a532a81db74ea8f7d5d4339f6b3e9f56  NbtAad.xml
bda26e70e469c324b01abcffd39978faba8d816380a51a8bbba1dfb49d1a1e15  NbtAad_ALL_Texts.zip
4a8412050e716d95d0162fb3c3c105b2c769734e531adb66bfad22ff2a00314f  NbtAad_ALL_Images.zip
```

Note that `NbtAad.xml` and `NbtAad_ALL_Texts.zip` differ from the source tree (they are rhmiapp-20's). The app must ship **these**, not the source tree's, or the menu will reference a text that doesn't exist.

`release.zip` also contains `NbtAad_Direct.xml` and `NbtAad_Multimedia.xml`, which are alternative menu-placement descriptors and are **not** part of the five-file install. Not shipped.

---

## 3b. Original blocker write-up (for the record)

This was the one needing your decision, now resolved by the section above.

`githubrepo/` is **source only**. The canonical install (`milestone-7/readme.md`, "Install") needs five files in `/opt/hmi/ID5/data/ro/Plugins/NbtAad/`:

| File | Present? |
|---|---|
| `libNbtAad.so` | **MISSING** |
| `NbtAad.manifest` | yes |
| `NbtAad.xml` | yes |
| `NbtAad_ALL_Texts.zip` | yes — 10 locales, zip valid |
| `NbtAad_ALL_Images.zip` | yes — 10 PNGs, zip valid |

`NbtAad.manifest` references it as `<file>{pluginDir}/NbtAad</file>`; HmiMain3D resolves that to `libNbtAad.so`, matching the Makefile's `-Wl,-soname,libNbtAad.so`.

Without that shared object there is **no Android Auto tile**. The daemon installs and runs, but nothing in Media/Radio launches it — which is the entire point of the install.

It cannot be produced here. Building it needs `qcc` from QNX SDP 6.5.0 (ARMv7LE, `_gpp` variant), a **QNX licence file**, and the Docker image `build.sh` expects. None of those exist on this machine, and none can run on the phone.

**Easy fix on your end:** `install.md` says the compiled artifacts are published on the Releases page as `release.zip`. Download the latest from

<https://github.com/Sq00ky/Supra-iDrive6-Android-Auto/releases>

and drop it in this folder. I'll verify its `libNbtAad.so` is ARM/QNX and built from this tree, the same way I did the daemon, then pin all five checksums.

Checksums of the four files we do have:

```
cc09123717fea5c41698c1ff4c08b4cd5dde620fda1b3d00b0e8699a3c621af2  NbtAad.manifest
be6be107316019eba1efafda51447bf7f4138ac14258fc3cf494b83fa3eb03e6  NbtAad.xml
36e5e7a0a8ff735108187f16a036a92dcf5fb336643c1d879e80b828c28cc62e  NbtAad_ALL_Texts.zip
4a8412050e716d95d0162fb3c3c105b2c769734e531adb66bfad22ff2a00314f  NbtAad_ALL_Images.zip
```

Treat these as provisional. If `release.zip` ships different versions of the XML and zips alongside its `.so`, the set must match the `.so`, not this source tree — the readme notes that XML and text-DB changes ride along with specific rhmiapp versions.

---

## 4. Other prerequisites the app cannot fix

All of these come from the repo, not from me:

1. **Apple CarPlay must be present and coded.** The daemon reuses the factory CarPlay Wi-Fi hotspot config (`iap2.cfg` / `iap2_Wifi.cfg`). `README.md` lists it as a hard requirement.
2. **One of `A4A`, `A4A_REMOTE_APPS`, `REMOTE_HMI` or `TOUCH_COMMAND` must be coded**, or HmiMain3D will not load PluginApps at all and the tile silently never appears. That is a BimmerCode job, which you already have — worth verifying before we touch anything, because otherwise a perfectly correct install looks like a failure.
3. **AAIDrive must not be installed or running** — known to break NBT-AAD.
4. **Legacy SSH crypto.** `install.md` connects with `-o HostKeyAlgorithms=+ssh-rsa -m hmac-sha1`, password `ts&SK412`. Most current Android SSH libraries disable `ssh-rsa` (SHA-1) and `hmac-sha1` by default, so the library choice is constrained — I need one where both can be explicitly re-enabled. Flagging it now because it is a dependency decision, not a detail.
5. **Address inconsistency in the repo's own docs.** `install.md` shows the proxy forwarding to `169.254.199.119:22` in one place and SSHing to `169.254.199.99` in another; your note says `.99`. The app will prefer the auto-detected address and only fall back to a default, so this resolves itself — but it is exactly why auto-detection matters more than a hardcoded address.
6. **Writable filesystem.** The install needs `mount -o remount rw /net/hu-omap/fs/sda0` and the `on -f /net/hu-omap/ /bin/sh` hop before anything copies. The backups manager has to record the pre-existing state of the `Plugins` directory — and since it does not exist on a stock unit, "restore" means *delete it again*. I'll record that explicitly rather than inferring it at restore time.

---

## 5. What comes next

Unblocked, can start immediately:

- Scaffold the Kotlin/Compose app: wizard shell, live monitor (HSFZ frames both directions plus SSH command/output, exportable), backups manager, Ethernet interface binding via `ConnectivityManager`/`bindProcessToNetwork`, foreground service with wake lock and screen-on, the 20% battery floor, and the `31 01 FD EE 38` stream filter.
- Finish the relay port: pin the discovery probe (UDP broadcast vs TCP sweep) and enumerate the relay tag numbering, then wire the car-to-server pipe.

No longer blocked (release.zip verified, see section 3):

- The daemon deploy plus HMI integration step, and its backup/restore path. Ships the six pinned files from `verified/`.
