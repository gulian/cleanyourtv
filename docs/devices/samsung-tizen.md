# Samsung Tizen: AU8000 reconnaissance and DU7000 custom-app testing

> **TL;DR: Don't spend time on this repo expecting to debloat a Samsung Tizen TV or replace
> Samsung Home. That workflow does not work here.** A separate custom Tizen app was tested, but
> the Home button still opened Samsung Home and a long-press restart returned to HDMI.
> Standby resume worked in one sequence; it is not a system launcher replacement.
> No safe system-app removal or full restorable backup was established.
> To turn off the tracking on a Samsung (or LG, Roku) TV with the remote instead, see
> [cleanyour.tv/not-android](https://cleanyour.tv/not-android).

One owner's reconnaissance from a Mac on the same network, not a port and not a tested list. It was
reported in [#441](https://github.com/Gathin23/cleanyourtv/issues/441) on 2026-10-20 by
[@csmith1133](https://github.com/csmith1133), with nothing changed on the TV. Nothing in `tools/`
speaks Tizen: `tools/tv.py` drives `adb`, and this set has no `adb` to connect to.

| Field | Observed value |
|---|---|
| Retail model | UN50AU8000FXZA, 50" AU8000, 2021 |
| Model string (`/api/v2/`) | `21_KANTSU2E_UHD_VOICE` |
| System | Tizen, US firmware |
| Network | Wi-Fi |

## AU8000: what answers without touching the TV

- SSDP replies as `Samsung-Linux/4.1, UPnP/1.0, Samsung_UPnP_SDK/1.0`, with a DMR on `9197` and
  `nservice` on `7678`.
- `GET http://<tv>:8001/api/v2/` returns the model, the power state, `TokenAuthSupport: true` and
  `developerMode: 0`.

| Port | State | What it is |
|---|---|---|
| 8001 | open | HTTP API and the remote-control websocket |
| 8002 | open | the same remote over TLS |
| 8080 | open | open in the report, purpose not identified |
| 9080 | open | open in the report, purpose not identified |
| 9197 | open | UPnP DMR (media rendering) |
| 7678 | open | `nservice` |
| 5555 | closed | no ADB on retail firmware |
| 26101 | closed | no `sdb` while Developer Mode is off |

## AU8000: what that rules out

- **No shell, so no `pm disable-user`.** Samsung's own apps cannot be disabled from outside on
  retail firmware, which is what every list in `examples/` is built on.
- **No screenshot.** `screencap` is part of that missing shell, and there is no capture API on 8001
  or 8002, so a script cannot see the TV to decide what the next key press should be.
- **No Home takeover.** No third-party app can take the Home button, so `launcher/` has nowhere to
  go here even if it could be installed.

## What a Tizen cleanup could still be

The remote on 8002 accepts key presses once the "Allow" prompt is accepted on the TV, and
`/api/v2/applications/<appId>` can launch an app, so a script can walk the settings menus the owner
named in the report: interest-based ads, viewing information services, autorun Smart Hub. That is a
key sequence with a person watching the screen, not a package list. Combined with DNS blocking at
the router, it is the realistic shape of a Tizen version of this repo: settings by remote keys plus
DNS, rather than debloat plus a launcher.

A port like that belongs in its own repository, with a row in [SHOWCASE.md](../../SHOWCASE.md), the
same as the [webOS launcher](https://github.com/OmkarK97/webos-vibehack).

## What the AU8000 report does not establish

- One set, one firmware, one region, one session. Tizen versions differ between years and markets,
  and none of this was checked on a second TV.
- Developer Mode was off, so the path through `sdb` on 26101 was not tested. A developer-mode set
  would still not have `adb`, and no `pm`-style package disabling was claimed.
- Whether the remote socket asks for the "Allow" prompt again after every TV reboot, and how long
  that grant lasts, is unmeasured.
- The app-launch endpoint was reported to work; which Smart Hub apps answer it, and whether it can
  reach a settings activity rather than only an app, is unmeasured.

If you have one of these sets and want to extend this page, a second unit on a different year or
region is the useful data point.

## DU7000: Developer Mode and a custom dashboard, tested on 2026-10-23

A second owner's session used a Samsung UA43DU7000UXEG, model string `24_KSUE_UB_T09`,
over Wi-Fi from an Apple Silicon Mac. SDB reported Tizen 9.0; the HTTP metadata API returned
`Unknown` for the firmware version, so the exact firmware build was not established.
This is a capability report, not a tested debloat list or a port of this repo's launcher.
No existing applications were removed, and no root or system-Home modification was attempted.

**The practical result:** a separately signed Tizen Web application could display a custom
full-screen interface, discover applications, launch selected applications and display a connected
HDMI source. It was not a replacement for Samsung Home. Distinguish these two goals before spending
time on the design.

### Network and developer access

These results are for the DU7000 with Developer Mode enabled and the Mac's address entered in the
Host PC IP field. A reachable port alone does not identify the service behind it.

| Port | DU7000 result | Verification | Limits |
|---|---|---|---|
| 8001 | open | HTTP `GET /api/v2/` returned model metadata | Remote-control websocket not tested |
| 8002 | open | HTTPS `GET /api/v2/` returned model metadata | Self-signed TLS certificate verification was bypassed for this local read only |
| 8080 | open | TCP connection | Service not identified |
| 9080 | closed or unreachable | TCP connection failed | Not distinguished further |
| 9197 | open | TCP connection | Service not identified independently |
| 7678 | open | TCP connection | Service not identified independently |
| 5555 | closed or unreachable | TCP connection failed | No ADB connection established |
| 26101 | open | SDB connected successfully | Developer Mode is not unrestricted shell or root access |

`sdb capability` reported `intershell_support:disabled`, `sdbd_rootperm:disabled`,
`rootonoff_support:disabled` and `filesync_support:pushpull`. An ordinary shell command did not
produce the expected output, and pulling `/etc/os-release` was refused with
`You cannot pull files from this path`. File-transfer support therefore did **not** establish
access to the TV's filesystem or a full backup.

The developer tooling could nevertheless install and debug a signed application. The absence of an
ordinary shell does not mean that the supported application-development path is unavailable.

### Mac setup and installation pitfalls

- Use Samsung's SDB, not Android's ADB. The tested SDB was version 4.2.36 from the official
  [Tizen Web CLI installer](https://download.tizen.org/sdk/Installer/tizen-sdk_10.0/).
  Its executable was Intel-only and ran through the Mac's existing Rosetta installation.
- The CLI installer downloaded about 326 MB. The installed CLI occupied about 478 MB, plus
  about 216 MB for its bootstrapped Java runtime, before additional SDK extensions.
  SDB itself was about 1 MB; that is not the size of the SDK installation.
- The first installer run failed while launching its bundled archive extractor, leaving no usable
  Java runtime. Extracting the installer's bundled JDK with macOS's `unzip` and rerunning the official
  installer using that runtime succeeded. No third-party SDB binary was used.
- Install the Samsung Certificate Extension and the TV deployment tools through the official
  Package Manager. The emulator was not needed. The TV-tools installation also brought in its
  resources dependency, including simulator resources.
- Do not run the graphical and command-line Package Managers concurrently. During an overlapping
  session a download failed because a partial file was missing; retrying after closing the graphical
  manager succeeded. The overlap was observed, not proven to be the cause.
- Follow Samsung's [certificate setup](https://developer.samsung.com/smarttv/develop/getting-started/setting-up-sdk/creating-certificates.html):
  sign in yourself, select TV and Public privileges, and include the target TV's DUID in the
  distributor certificate. Keep certificate files, passwords and signing profiles private.
- In Device Manager, use **Permit to install applications** for the connected TV.
- **Use a package filename without spaces.** Both CLI and Device Manager attempts with a
  space-containing filename failed without an actionable error. The owner successfully installed
  after using a space-free filename. Earlier failures did not establish a TV-side signature rejection.
- Running the CLI from the package's directory, with no trailing `-- <directory>`, worked for the
  owner. Use a separate application identity so the test cannot overwrite an existing application.

For example, from the directory containing your own signed package:

```sh
sdb connect <tv ip>:26101
tizen install -n DashboardTest.wgt -s <tv ip>:26101
```

This is a developer-app workflow, not permission to remove system components or bypass verification.
Keep the source and signing materials backed up locally before updating the test.

### What the custom application actually established

| Capability | Observed result |
|---|---|
| Full-screen Web application and physical D-pad navigation | Worked |
| `tizen.application.getAppsInfo()` | Returned 486 API-visible entries, including system and helper entries |
| Launching the three owner-selected streaming applications | Owner confirmed the manual tests worked |
| Source discovery with `tizen.systeminfo.getPropertyValue("VIDEOSOURCE")` | Returned connected and disconnected sources |
| Connected HDMI 1 via `tizen.tvwindow` | Owner confirmed the manual picture test worked |
| Disconnected HDMI 3 | Diagnostic deliberately disabled its test button; switching this port was not tested |
| Physical Home button while the diagnostic was open | Opened Samsung Home |

Samsung documents source selection in the [TV Window API](https://developer.samsung.com/smarttv/develop/api-references/tizen-web-device-api-references/tvwindow-api.html)
as a **public** capability requiring `http://tizen.org/privilege/tv.window`.
Do not assume that HDMI control requires root. This test displayed HDMI inside the application's TV
window; it did not establish every aspect of a system-wide source handoff.

The [Application API](https://developer.samsung.com/smarttv/develop/api-references/tizen-web-device-api-references/application-api.html)
allowed inventory and selected-app launch tests. An entry being visible is not evidence that it is
user-installed, removable, safe to remove, or running in the background.

### Startup: standby resume is not a Home replacement

On this Tizen 9 interface, the menu was under **All Settings > Advanced features > Start Screen
Options**, rather than the older General & Privacy location. Before testing, the owner photographed
the original values: **Start with Smart Hub Home on**, **Autorun Last App off**. Those are this unit's
undo values, not recommended defaults for every TV.

The owner then tested with Smart Hub startup off and last-app autorun on:

| Test | Observed result |
|---|---|
| Normal short-press standby off/on after an HDMI test | Returned to HDMI 1, despite the diagnostic interface having been visible |
| Normal standby off/on after freshly opening the diagnostic without an HDMI test | Reopened the diagnostic |
| Long-press Power restart | Returned to HDMI 1 |
| Physical Home button | Opened Samsung Home |

This establishes **standby resume in one tested sequence**, not guaranteed startup or Home takeover.
Samsung [warns that not every application supports last-app autorun](https://www.samsung.com/ae/support/tv-audio-video/use-autorun-start-screen-options-on-your-samsung-tv/).
Power removal/reconnection, repeated restart trials and startup after using other applications were
not tested. No supported way to replace the physical Home button's destination was established.

### Inventory export, backup limits and removal

The installed diagnostic was opened in debug mode through SDB. Its Web Inspector exposed a local
debugging endpoint; evaluating `tizen.application.getAppsInfo()` within that application allowed the
inventory to be saved on the Mac as JSON and CSV. Both exports contained 486 unique application IDs.
The API did **not** expose a removable flag in this session; the export recorded it as unknown.
This avoided installing an inventory collector or publishing the owner's application list.

The metadata, capability results, signing materials and original startup-settings photo were
preserved privately on the Mac. **None is a full, restorable TV backup.** App data, accounts,
system settings and firmware were not backed up, and system-package reinstallation was not verified.

No uninstall experiment was performed. In particular:

- An ad-related name does not establish that a component is safe to remove or that other TV features
  do not depend on it.
- Removing a TV advertising component would not establish removal of advertisements delivered inside
  streaming applications.
- A large inventory count does not measure background resource use. No speed, memory, storage or
  ad-reduction improvement was measured.

Do not treat this report as an uninstall list. A custom dashboard can limit its own visible choices
without removing applications or risking HDMI, remote-control or streaming functionality.
The custom Tizen application belongs in a separate repository; this note does not add Tizen support
to `tools/` or make the Android launcher installable on Samsung firmware.

## NU7405 (2018): the same picture on an older set, checked on 2026-10-24

A third owner's read-only check from a Mac on the same network, with nothing changed on the TV and
Developer Mode off. Only `GET /api/v2/` on 8001 and TCP connections were used.

| Field | Observed value |
|---|---|
| Retail model | UE50NU7405, 50" NU7400 series, 2018, Spain |
| Model string (`/api/v2/`) | `18_KANTME_UHD` |
| System | Tizen; the API returned `Unknown` for the firmware version |
| API | `version` 2.0.25, `TokenAuthSupport: true`, `developerMode: 0` |
| Network | Wi-Fi |

| Port | NU7405 result | Verification | Limits |
|---|---|---|---|
| 8001 | open | HTTP `GET /api/v2/` returned model metadata | Remote-control websocket not tested |
| 8002 | open | TCP connection | TLS endpoint not read |
| 8080 | open | TCP connection | Service not identified |
| 9080 | closed | Connection refused | Open on the AU8000 |
| 9197 | open | TCP connection | Service not identified independently |
| 7678 | open | TCP connection | Service not identified independently |
| 5555 | closed | Connection refused | No ADB on retail firmware |
| 26101 | closed | Connection refused | Developer Mode off |

Three years older than the AU8000, the set answers the same way: no ADB, no shell and no `sdb`, so
the lists in `examples/` and the launcher do not apply to it either. The TV did not answer SSDP
discovery from the Mac and was found only by scanning the subnet for 8001, 9197 and 7678.
The `/api/v2/` reply also carries the DUID, the Wi-Fi MAC, the access point's MAC, the IP address and
the name given to the TV; leave those out of anything you publish.

## Q67BA (2022): the useful data point, one region and one year over, checked on 2026-10-24

A fourth owner's read-only check on a fourth set, a year on from the AU8000 and in a different
region, with nothing changed on the TV and Developer Mode off, reported in
[#441](https://github.com/Gathin23/cleanyourtv/issues/441) on 2026-10-24 by
[@niepiekm](https://github.com/niepiekm). Only `GET /api/v2/` on 8001 and 8002, TCP connections on
the eight ports already documented, and one SSDP probe were used. Nothing here checks a second
Q67BA or a post-reboot state, and the set was one unit, one session, one read pass.

| Field | Observed value |
|---|---|
| Retail model | QE43Q67BAUXXH, 43" Q67BA (QLED Q6 series), 2022, Poland |
| Model string (`/api/v2/`) | `22_NIKEL_UHD` |
| System | Tizen; the API returned `Unknown` for the firmware version |
| API | `version` 2.0.25, `TokenAuthSupport: true`, `developerMode: 0`, `PowerState: on` |
| Network | Wi-Fi |

| Port | Q67BA result | Verification | Limits |
|---|---|---|---|
| 8001 | open | HTTP `GET /api/v2/` returned model metadata | Remote-control websocket not tested |
| 8002 | open | HTTPS `GET /api/v2/` returned metadata matching 8001 in every device field | Self-signed TLS certificate verification was bypassed for this local read only |
| 8080 | open | TCP connection | Service not identified |
| 9080 | open | TCP connection | Service not identified |
| 9197 | open | TCP connection | Consistent with the AU8000's UPnP DMR; not identified independently here |
| 7678 | open | TCP connection | Consistent with the AU8000's `nservice`; not identified independently here |
| 5555 | closed | No response | No ADB on retail firmware |
| 26101 | closed | No response | Developer Mode off; `sdb` therefore not installed and not tested |

The TV answered on 8001 during the same minute window in which an SSDP M-SEARCH (`ST: ssdp:all`)
sent once produced no reply. That matches the NU7405 and diverges from the AU8000; only one probe
was sent, so an unicast or repeated-match retry is untested. Polling 8001 is the reliable way to
find this set.

Across the four sets on this page, the Q67BA adds two data points:

- **9080** is now observed open on a 2021 unit (AU8000) and on this 2022 unit, and closed on the
  2018 and 2024 units. The service remains unidentified, and four observations are too few to read
  a generational rule from.
- **`firmwareVersion: Unknown`** is now confirmed across four TV years and three regions, which
  looks like a Samsung API behaviour rather than a per-model defect.

What carries over unchanged: no ADB, no shell on retail firmware, and with `developerMode: 0` no
`sdb`, so the developer-mode path (signed custom app, debug install, Web Inspector) was neither
available nor tested on this generation. `TokenAuthSupport: true` is an API-declared flag only: no
remote-control session was opened, so no "Allow" prompt was triggered, no pairing grant exists from
this session, and grant duration remains unmeasured on this set. The `/api/v2/` reply carried the
DUID, the Wi-Fi MAC and the access point's MAC, which are left out of everything published here.

## LS01T "The Serif" (2020): Developer Mode on a 2020 set, an app inventory over `sdb`, and a grant that lasts, checked on 2026-10-28

A fifth owner's check on a fifth set, between the NU7405 and the AU8000 in years and in a fourth
region, from a Linux host on the same network, with nothing changed on the TV. The difference from
the other units is that Developer Mode has been on since 2026-08-22, when a signed Web app of the
owner's was installed through Samsung's developer tooling, so `sdb` could be asked what it allows.
Only `GET /api/v2/` on 8001 and 8002, TCP connections, one SSDP probe, HTTP reads of what answered,
and `sdb` commands that list things were used. Where this section cites the August install, it says so.

| Field | Observed value |
|---|---|
| Retail model | QE50LS01TAUXXC, 50" The Serif (LS01T), 2020, France |
| Model string (`/api/v2/`) | `20_NIKEM_SERIF` |
| System | Tizen; `sdb capability` reports `platform_version:5.5`, the API returned `Unknown` for the firmware version |
| API | `version` 2.0.25, `TokenAuthSupport: true`, `developerMode: 1`, `FrameTVSupport: false`, `PowerState: on` |
| Network | Wi-Fi |

| Port | Serif result | Verification | Limits |
|---|---|---|---|
| 8001 | open | HTTP `GET /api/v2/` returned model metadata | |
| 8002 | open | HTTPS `GET /api/v2/` returned metadata matching 8001 in every device field; the remote websocket accepted a token issued in August | Self-signed TLS certificate verification was bypassed for this local read only |
| 8080 | open | HTTP `GET /ws/app/YouTube` returned a DIAL 2.2 application descriptor | The DIAL application server, see below |
| 9080 | closed | Connection refused | Closed on the 2018, 2020 and 2024 units, open on 2021 and 2022 |
| 9197 | open | HTTP `GET /` answered `400` with `Server: UPnP/1.1 Samsung AllShare Server/1.0`; SSDP advertises `/dmr` here | UPnP media renderer, consistent with the AU8000 |
| 7678 | open | HTTP `GET /nservice/` returned a UPnP device description | The DIAL receiver, see below |
| 5555 | closed | Connection refused | No ADB on retail firmware |
| 26101 | open | `sdb connect` succeeded | Developer Mode on; what `sdb` can do is below |

Two of the ports left unidentified on the other units name themselves here. `GET /nservice/` on
7678 is a UPnP device description of type `urn:dial-multiscreen-org:device:dialreceiver:1` whose
`Application-URL` header is `http://<tv ip>:8080/ws/app/`, and **8080 is that DIAL application
server**: `GET /ws/app/<name>` answered `200` with a `dialVer="2.2"` descriptor (name, state, version)
for `YouTube`, `Netflix` and the owner's app, behind a bare `WebServer` header. The 7678 description also
carries a `<sec:deviceID>` with the exact shape of a DUID that is **not** the DUID: a certificate issued
on that value installed nothing in August, and only `sdb shell "0 getduid"` gives the real one. The
one SSDP M-SEARCH got fourteen replies, like the AU8000 and unlike the NU7405 and Q67BA, from the
renderer on 9197 and from a ninth port, **9119**, a `urn:samsung.com:device:ScreenMirroring:1` device
at `/screen_sharing`. The `/api/v2/`, `/nservice/` and `/screen_sharing` replies all carry identifiers
(DUID, UDN, serial, MACs, the TV's name, and the `developerIP` field once Developer Mode is on); none is
reproduced here.

Polling 8001 finds this set only when it is on. In standby it still answers ARP and Wake-on-LAN, but
all eight ports time out rather than refusing, so a probe hangs for its whole timeout instead of
failing fast. A burst of magic packets, unicast and broadcast, repeated, had `GET /api/v2/` answering
within three seconds today; a single packet was seen in August to leave it half-woken, panel off and
remote keys ignored. Its address had also moved since August, and it was found again by its MAC in
the ARP table, not at the old address.

`sdb` (4.2.16, from the `egorshulga/tizen-sdk` container image, no Tizen Studio on the host) reports
the same closed doors on this Tizen 5.5 as the DU7000's Tizen 9: `intershell_support:disabled`,
`sdbd_rootperm:disabled`, `rootonoff_support:disabled`, `filesync_support:pushpull`,
`appcmd_support:disabled`, `cpu_arch:armv7`. `sdb shell "cat /etc/os-release"` and `sdb shell "ls /"`
print nothing, and `sdb pull /etc/os-release` is refused with the DU7000's `You cannot pull files from
this path`. What it does give, with nothing installed, is **`sdb shell "0 applist"`**: the installed
applications for the TV user, name and app ID, **394 entries** here, the store apps, the Internet
browser (`com.samsung.tv.knox-browser`), the owner's app and Samsung's own services (SmartThings,
Universal Guide, the CoBA home components, HDMI-CEC, an autofill service, a YouTube test suite). The
DU7000 needed a custom app and `tizen.application.getAppsInfo()` for the same kind of list. It carries
no removable flag, nothing here says any entry is safe to touch, and the list itself stays with the
owner. Two things about Developer Mode that cost time in August: **26101 stays closed until the TV is
restarted** after the mode is enabled, and the Host PC IP field has to hold the address of the machine
that will run `sdb`.

The remote-control grant lasts. The token this set issued on 2026-08-22 opened the websocket on 8002
today with `ms.channel.connect` after 1.5 s and no prompt on screen, through the Developer Mode
restart, standby cycles and the address change: sixty-seven days is a lower bound, not the limit.
The prompt itself closes after about 30 s (`ms.channel.timeOut`), so a first pairing wants a retry
rather than one attempt. On the REST side, `GET /api/v2/applications/<appId>` returns `{running,
visible, version}` for an installed app and `404` for an absent one, which is the only installation
check without `sdb`: the websocket `ed.installedApp.get` request is mute on this firmware, as is
`ed.apps.launch`. `visible` is the top of the application stack, not what the panel shows. In August,
`POST` launched an app and `DELETE` took it off the screen but left it `running: true`; the Internet
app launched but ignored a URL passed as `metaTag` or in the request body, checked against a web server
that never saw a request; and DIAL YouTube with content answered `403`.

The August install itself adds four pitfalls to the DU7000's list, from Linux with the Tizen CLI in
the `egorshulga/tizen-sdk` container and no Tizen Studio GUI. Samsung's distributor certificate, with
the set's DUID in it, is mandatory; the generic Tizen distributor profile is refused at install, and
adding a set later meant reissuing it. The Java CLI rejects PKCS#12 files encrypted with PBES2/AES
and calls them `Invalid file or password`, which they are not; legacy `PBE-SHA1-3DES` with the same key
works. A space in config.xml's `<name>` fails the install with no message, the twin of the DU7000's
space in the filename: `<name>TestApp</name>` installed, `<name>Test App</name>` did not. And the default
content security policy blocks inline `<style>` and `<script>`, a black screen with a white square until
a `<tizen:content-security-policy>` entry lifts it. The Home button was not touched and opens Samsung
Home, as on the DU7000.

What this does not establish: one set, one firmware, one session, and the `sdb` findings come from a
set whose Developer Mode was already on. No app was removed or disabled, no debloat was attempted, and
`0 applist` says nothing about what the 394 entries do. The page's conclusion stands on a fifth set:
no shell, no package disabling, no Home takeover.

## LS03T "The Frame" (2020): the same year as the Serif, with `FrameTVSupport: true`, checked on 2026-10-28

The same owner's third set, a 65" Frame of the Serif's year, read the same day in the same way, with
nothing changed on it. Developer Mode has been on since 2026-08-22, for the same signed app. Only
`GET /api/v2/` on 8001 and 8002, TCP connections, one SSDP probe, HTTP reads of the DIAL and UPnP
descriptions, `sdb capability` and `sdb shell "0 applist"` were used; the one Frame-specific
behaviour below comes from August and is marked as such.

| Field | Observed value |
|---|---|
| Retail model | QE65LS03TAUXXC, 65" The Frame (LS03T), 2020, France |
| Model string (`/api/v2/`) | `20_NIKEM_FRAME` |
| System | Tizen; `sdb capability` reports `platform_version:5.5`, the API returned `Unknown` for the firmware version |
| API | `version` 2.0.25, `TokenAuthSupport: true`, `developerMode: 1`, **`FrameTVSupport: true`**, `PowerState: on` |
| Network | Wi-Fi |

| Port | Frame result | Verification | Limits |
|---|---|---|---|
| 8001 | open | HTTP `GET /api/v2/` returned model metadata | |
| 8002 | open | HTTPS `GET /api/v2/` returned metadata matching 8001 in every device field; the remote websocket accepted a token issued in August | Self-signed TLS certificate verification was bypassed for this local read only |
| 8080 | open | `GET /ws/app/YouTube` and `/ws/app/Netflix` returned DIAL 2.2 descriptors; `GET /` is `404` | DIAL application server, as on the Serif |
| 9080 | closed | Connection refused | Closed on the 2018, 2020, 2024 and 2025 units, open on 2021 and 2022 |
| 9197 | open | `GET /` answered `400` with `Server: UPnP/1.1 Samsung AllShare Server/1.0` | UPnP media renderer |
| 7678 | open | `GET /nservice/` returned the DIAL receiver description, `Application-URL` pointing at 8080 | |
| 5555 | closed | Connection refused | No ADB on retail firmware |
| 26101 | open | `sdb connect` succeeded | Developer Mode on |

Everything the Serif answered, this set answers the same way: the DIAL and UPnP layout, 9119 open
with the `ScreenMirroring:1` description, fourteen SSDP replies, `sdb` with `intershell_support:disabled`,
`sdbd_rootperm:disabled`, `rootonoff_support:disabled`, `appcmd_support:disabled` and the pull refused,
`sdb shell "0 applist"` with **393 entries** (the Serif's 394, with the Frame's own art components,
`org.tizen.art-app` and the CoBA art entries, in place of a few others), the same REST IDs for YouTube,
Internet and Plex, and the token issued on 2026-08-22 accepted today with `ms.channel.connect` in 1.5 s
and no prompt: sixty-seven days, through the Developer Mode restart and an address change.

What the Frame adds is the power key, observed in August with the owner watching the panel and the
API read back after each key. A short `KEY_POWER` over the remote websocket does **not** turn this set
off: it closes the application in front, `PowerState` stays `on` and the panel stays lit, which from
outside looks like an ignored command. A `KEY_POWER` held for about three seconds turned it off, in
about fifteen seconds to `PowerState: standby`; Wake-on-LAN brought it back in about seven. The set
says what it is itself, through `FrameTVSupport` in `/api/v2/`, so a tool does not need a model list to
pick the long press. Nothing was pressed today. Two readings that apply to every set on this page
came from the same August sessions: the embedded HTTP server does not take parallel requests well
(four `GET /api/v2/applications/<id>` probes sent at once lost three of forty-eight to timeouts; sent
in series, none), and `visible` reports the top of the application stack, not the panel: an app left
open while broadcast TV was watched still read `visible: true` here.

What this does not establish: one set, one session; the art mode itself (its own websocket channel
exists) was not queried, the long-press figures are from one August run, and no Frame of another year
was checked.

## LSM7F "The Movingstyle" (2025): Tizen 9 in Europe, checked on 2026-10-28

The same owner's 2025 set, a 27" portable screen on a battery, read the same day in the same way,
with nothing changed on it. Developer Mode has been on since 2026-08-26, for the same signed app as
the Serif. Only `GET /api/v2/` on 8001 and 8002, TCP connections, one SSDP probe, HTTP reads of the
DIAL and UPnP descriptions, `sdb capability` and `sdb shell "0 applist"` were used.

| Field | Observed value |
|---|---|
| Retail model | UE27LSM7FAXXXE, 27" The Movingstyle (LSM7F), 2025, France |
| Model string (`/api/v2/`) | `25_PTM_MSC` |
| System | Tizen; `sdb capability` reports `platform_version:9.0`, the API returned `Unknown` for the firmware version |
| API | `version` 2.0.25, `TokenAuthSupport: true`, `developerMode: 1`, `FrameTVSupport: false`, `resolution` 2560x1440, `PowerState: on` |
| Network | Wi-Fi |

| Port | Movingstyle result | Verification | Limits |
|---|---|---|---|
| 8001 | open | HTTP `GET /api/v2/` returned model metadata | |
| 8002 | open | HTTPS `GET /api/v2/` returned metadata matching 8001 in every device field; the remote websocket accepted a token issued in August | Self-signed TLS certificate verification was bypassed for this local read only |
| 8080 | open | `GET /ws/app/YouTube` returned a DIAL 2.2 descriptor; `GET /` is `403` | DIAL application server, as on the Serif |
| 9080 | closed | Connection refused | Closed on the 2018, 2020, 2024 and 2025 units, open on 2021 and 2022 |
| 9197 | open | `GET /` answered `400` with `Server: UPnP/1.1 Samsung AllShare Server/1.0` | UPnP media renderer |
| 7678 | open | `GET /nservice/` returned the DIAL receiver description, `Application-URL` pointing at 8080 | |
| 5555 | closed | Connection refused | No ADB on retail firmware |
| 26101 | open | `sdb connect` succeeded | Developer Mode on |

Port for port, this Tizen 9 of 2025 answers like the Tizen 5.5 of 2020: the same DIAL and UPnP
layout, 9119 open with the same `ScreenMirroring:1` description, fourteen SSDP replies (the DIAL
receiver on 7678 advertises itself here too), the same `Samsung-Linux/4.1` server string, and `sdb`
with `intershell_support:disabled`, `sdbd_rootperm:disabled`, `rootonoff_support:disabled`,
`appcmd_support:disabled` and the pull refused. `sdb shell "0 applist"` works here as well and lists
**510 entries**, in the range of the 486 the DU7000 owner read through `getAppsInfo()`.

Three differences. The set leaves the network when it is really off: on battery it powers down rather
than standing by, and then answers neither ARP nor Wake-on-LAN, so no scan finds it (the owner first
located it through the Samsung account's device list). Today it was in standby, present in ARP with
every port timing out, and a magic-packet burst had 8001 answering at once. The numeric app IDs are not
portable across years: the REST ID that is the Internet app on the 2020 set is `404` here, while
`0 applist` lists the browser as `com.samsung.tv.knox-browser` ("Workspace Browser"); `GET
/api/v2/applications/<id>` still returns `{running, visible, version}` for YouTube and Plex and `404`
for an app the set does not have, which DIAL agrees with (`/ws/app/Netflix` is `404` here, `200` on
the Serif). And `/api/v2/` carries two fields the 2020 set lacks, `EdgeBlendingSupport` and
`mouseWheelSupport`, with a `WallScreenRatio` of `-1` rather than `0`.

The token this set issued on 2026-08-26 was accepted today with `ms.channel.connect` in 0.2 s and no
prompt: sixty-three days, through complete power-offs and an address change, a second data point
under the Serif's and the Frame's sixty-seven.

Across the seven sets on this page: 9080 is closed on every unit outside 2021 and 2022, and still
unidentified; `firmwareVersion: Unknown` holds on six years and four regions; 8080 and 7678 are DIAL
on the three sets where they were read, and 9119 answers on all three; Developer Mode gives a signed app and an
inventory, never a shell; and a remote grant has been seen to last two months on three sets. One owner,
one session, three sets: none of it was checked on a second unit of either model.
