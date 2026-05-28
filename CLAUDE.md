# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

GNOME Network Displays is an experimental Wi-Fi Display (Miracast) and Chromecast
sender for GNOME. It is written in C using GObject, GTK4/libadwaita, and GStreamer,
and built with Meson. It streams a monitor/window (captured via the XDG screencast
portal, falling back to X11) to a remote sink over WFD (RTSP) or Chromecast.

## Build & run

```sh
# Install build deps (Fedora): dnf builddep gnome-network-displays
meson setup _build
meson compile -C _build
meson install -C _build          # optional

# Regenerate the translation template (CI fails if this is out of date):
meson compile -C _build gnome-network-displays-pot
```

There is **no automated test suite**. Verification is manual:

```sh
# Run with a fake local sink instead of real hardware. Then connect any RTSP
# client to rtsp://localhost:7236/wfd1.0
NETWORK_DISPLAYS_DUMMY=1 G_MESSAGES_DEBUG=all ./_build/src/gnome-network-displays
```

Debugging env vars: `G_MESSAGES_DEBUG=all` (the GLib log domain is `Gnd`),
`GST_DEBUG=*:5` for GStreamer pipelines, `GST_DEBUG_DUMP_DOT_DIR=<dir>` to dump
pipeline graphs. See README.md for the (lengthy) P2P/connection debugging guide.

### Meson options (`meson_options.txt`)
- `systemd_resolved` (default **false**): when false, mDNS discovery uses Avahi
  (`nd-*-provider.c`); when true, it uses systemd-resolved (`nd-sd-*-provider.c`).
  This choice is made at build time and swaps which source files compile (see
  `src/meson.build`).
- `build_app` / `build_daemon`: toggle the GUI app and the headless daemon.
- `firewalld_zone`: installs the `P2P-WiFi-Display` firewalld zone (`data/`).

### Code style
GNU/GNOME style enforced by uncrustify. Run `scripts/uncrustify.sh` (uses
`scripts/uncrustify.cfg`) before committing C changes.

### Commit messages
Commit message language: English (matches the upstream GNOME history).

## Architecture

### Three executables, shared core
All three link the `wfd-server` and `cc-cast-channel` static libraries and share
`gnome_nd_common_sources` (the provider/sink layer). Defined in `src/meson.build`:
- **`gnome-network-displays`** (`src/app/`) — the GTK4/libadwaita GUI. Runs everything
  in-process: discovery, capture, and the WFD RTSP server.
- **`gnome-network-displays-daemon`** (`src/daemon/`) — a headless D-Bus service
  (`org.gnome.NetworkDisplays.Daemon`) exposing the `org.gnome.NetworkDisplays.Manager`
  interface (`src/org.gnome.NetworkDisplays.Manager.xml`).
- **`gnome-network-displays-stream`** (`src/stream/`, installed to libexec) — a
  per-stream helper. The daemon does **not** stream itself; on `StartStream` it spawns
  one stream process per sink as a systemd **transient unit**
  (`gnome-network-displays-stream-<uuid>.service`) over `org.freedesktop.systemd1`.
  See `nd-manager.c` and `nd-systemd-helpers.c`.

### Provider / Sink abstraction (the central design)
Two GObject interfaces decouple discovery from the UI/streaming code:
- **`NdProvider`** (`nd-provider.h`) — discovers sinks; emits `sink-added`/`sink-removed`.
- **`NdSink`** (`nd-sink.h`) — a streamable target with `start_stream`/`stop_stream`/`to_uri`,
  a state machine (`NdSinkState`: `DISCONNECTED → ENSURE_FIREWALL → WAIT_P2P →
  WAIT_SOCKET → WAIT_STREAMING → STREAMING`, plus `ERROR`), and a protocol tag
  (`NdSinkProtocol`: WFD_P2P, WFD_MICE, CC, plus DUMMY/META variants).

`NdMetaProvider` aggregates all concrete providers into one list; `NdMetaSink`
de-duplicates sinks that are the same physical device reachable via multiple
protocols, picking the highest-priority one. The GUI (`nd-window.c`) and the daemon
(`nd-daemon.c`) both build an `NdMetaProvider` and register the same set of providers
into it.

Concrete providers/sinks (in `src/`):
- **WiFi-Direct P2P** — `NdNMDeviceRegistry` (`nd-nm-device-registry.c`) watches
  NetworkManager (libnm) for P2P-capable Wi-Fi devices and creates
  `nd-wfd-p2p-provider` / `nd-wfd-p2p-sink`.
- **WFD MICE** (WFD over infrastructure network, mDNS-discovered) — `nd-wfd-mice-*`.
- **Chromecast** — `nd-cc-*`.
- **Dummy** — `nd-dummy-*`, enabled only with `NETWORK_DISPLAYS_DUMMY=1`.

### Streaming backends
- **WFD** (`src/wfd/`, the `wfd-server` static lib): `WfdServer` subclasses
  `GstRTSPServer`. `wfd-media-factory`/`wfd-media` build the GStreamer RTSP pipeline;
  `wfd-params`/`wfd-video-codec`/`wfd-audio-codec`/`wfd-resolution` negotiate WFD
  capabilities with the sink; `wfd-client` and `wfd-session-pool` manage RTSP sessions.
- **Chromecast** (`src/cc/`, the `cc-cast-channel` static lib): `cast_channel.proto`
  (compiled by protobuf-c into `cast_channel.pb-c.{c,h}`) is the Cast wire protocol.
  `cc-comm` is the TLS channel, `cc-ctrl` drives the Cast control/state machine,
  `cc-http-server` serves media to the device, `cc-media-factory` builds the pipeline.
- **Capture & audio**: the GUI uses libportal's screencast portal → `pipewiresrc`,
  falling back to X11 `ximagesrc` (`nd-window.c`). Audio is captured from the
  PulseAudio "Network-Displays" sink (`nd-pulseaudio.c`).

### Generated code (don't hand-edit the outputs)
Meson generates several sources at build time: `gdbus_codegen` for the two D-Bus
`.xml` interfaces (`NdDBus*`), `mkenums` for the `NdSink*` enums in `nd-sink.h`
(`nd-enum-types`), protobuf-c for `cast_channel`, and `gresource` for the UI
(`gnome-network-displays.gresource.xml`, including `nd-window.ui`).
