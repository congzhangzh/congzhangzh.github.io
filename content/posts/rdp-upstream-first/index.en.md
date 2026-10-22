---
title: "Remote desktop on Linux, upstream-first: AV1, Wayland and a browser client"
date: 2026-10-22
summary: "What we are building across FreeRDP, xrdp and GNOME Remote Desktop: AV1 from server to browser, Wayland sessions in xrdp, and reproducible builds anyone can try today. Every change goes upstream first."
tags: ["rdp", "freerdp", "xrdp", "gnome-remote-desktop", "av1", "wayland", "webassembly"]
---

Remote desktop on Linux has long meant X11 sessions, older codecs and a client you have to
install. We want something else: an open-source stack that is **fast** (AV1 end to end),
works with **Wayland** desktops (GNOME, KDE Plasma, sway), and opens in **a browser tab**
with nothing to install.

None of this lives in a private fork. Every change goes to the upstream project as a pull
request (FreeRDP, xrdp) or merge request (GNOME Remote Desktop), and the builds we publish
are made from upstream sources only. This post is the map: what exists, what has landed, and
what is still in review. Numbers are as of 22 October 2026.

## The pieces

```
 browser tab                         RDP server on Linux
┌────────────────────────┐          ┌──────────────────────────────────────┐
│ FreeRDP SDL3 client    │   RDP    │ xrdp ── wlup ── GNOME / KDE / sway   │
│ compiled to WebAssembly│◀────────▶│   (Wayland session started by sesman)│
│ AV1/H.264: WebCodecs   │          │                                      │
│ sound, mic: Web Audio  │          │ or gnome-remote-desktop (GNOME's own)│
└────────────────────────┘          └──────────────────────────────────────┘
          ▲                                          ▲
          └──── rdp-release-kit: reproducible builds of both ends ────┘
```

| Project | Our pull/merge requests | Status |
|---|---|---|
| [FreeRDP](https://github.com/FreeRDP/FreeRDP/pulls?q=author%3Acongzhangzh) | 45 since 30 September 2026 | 33 merged, 6 in review |
| [xrdp](https://github.com/neutrinolabs/xrdp/pulls?q=author%3Acongzhangzh) | 9 | 1 merged, 8 in review (including the Wayland RFC) |
| [pulseaudio-module-xrdp](https://github.com/neutrinolabs/pulseaudio-module-xrdp/pull/143) | 1 | in review |
| [GNOME Remote Desktop](https://gitlab.gnome.org/GNOME/gnome-remote-desktop/-/merge_requests?author_username=congzhangzh) | 6 | in review |

## 1. FreeRDP in the browser

[FreeRDP Web](https://congzhangzh.github.io/freerdp-web/) runs FreeRDP's own SDL3 client,
compiled to WebAssembly with Emscripten. It is not a separate JavaScript reimplementation of
RDP: it is the same C code as the desktop client, so every protocol fix helps both.

The interesting part is what the browser does for us. Decoding AV1 in WebAssembly would cost
a lot of CPU, but browsers already contain video decoders, often hardware-accelerated, and
expose them through the **WebCodecs** API. So the client hands each compressed frame to
WebCodecs, and the browser decodes it, on the GPU where it can. (Tested so far: Chrome.)

Upstream so far:

- Merged: an Emscripten build and unit-test job in FreeRDP's CI
  ([#13728](https://github.com/FreeRDP/FreeRDP/pull/13728)), the build fixes it needed
  ([#13724](https://github.com/FreeRDP/FreeRDP/pull/13724)),
  **AV1 decoded with WebCodecs** ([#13731](https://github.com/FreeRDP/FreeRDP/pull/13731)),
  **sound through Web Audio** ([#13732](https://github.com/FreeRDP/FreeRDP/pull/13732)) and
  an Opus microphone encoding fix ([#13810](https://github.com/FreeRDP/FreeRDP/pull/13810)).
- In review: the browser clipboard ([#13733](https://github.com/FreeRDP/FreeRDP/pull/13733)),
  a Web Audio microphone ([#13811](https://github.com/FreeRDP/FreeRDP/pull/13811)) and
  H.264 through WebCodecs ([#13812](https://github.com/FreeRDP/FreeRDP/pull/13812)).

## 2. FreeRDP on the desktop

Working on the browser client kept turning up issues that every FreeRDP user hits, so many
of the merged changes are for the native clients on Linux, Windows and macOS:

- **AV1:** correct colours for 4:4:4 frames
  ([#13561](https://github.com/FreeRDP/FreeRDP/pull/13561)), and linking dav1d and aom when
  pkg-config can't find them ([#13625](https://github.com/FreeRDP/FreeRDP/pull/13625)).
- **Sound:** smoother playback on PulseAudio, Windows and macOS
  ([#13722](https://github.com/FreeRDP/FreeRDP/pull/13722),
  [#13723](https://github.com/FreeRDP/FreeRDP/pull/13723),
  [#13727](https://github.com/FreeRDP/FreeRDP/pull/13727),
  [#13574](https://github.com/FreeRDP/FreeRDP/pull/13574)).
- **Clipboard and files:** images and UTF-8 text on Windows
  ([#13633](https://github.com/FreeRDP/FreeRDP/pull/13633)), copy and paste of files with the
  macOS Finder ([#13591](https://github.com/FreeRDP/FreeRDP/pull/13591)), and drive
  redirection against FreeRDP-based servers
  ([#13592](https://github.com/FreeRDP/FreeRDP/pull/13592),
  [#13593](https://github.com/FreeRDP/FreeRDP/pull/13593)).
- **Resizing and monitors:** codecs reset on reactivation
  ([#13668](https://github.com/FreeRDP/FreeRDP/pull/13668)), layouts resent to a new display
  channel ([#13726](https://github.com/FreeRDP/FreeRDP/pull/13726)), and desktop vs device
  scale kept apart ([#13806](https://github.com/FreeRDP/FreeRDP/pull/13806)).

## 3. xrdp on Wayland

xrdp is the RDP server most Linux users reach for, and today it serves X11 sessions. The RFC
[neutrinolabs/xrdp#3904](https://github.com/neutrinolabs/xrdp/pull/3904) adds a new backend
module, **wlup**, and two session types that sesman starts like any other:

| | sway (wlroots) | GNOME (Mutter) |
|---|---|---|
| frames | `ext-image-copy-capture-v1`, damaged regions only | ScreenCast → PipeWire |
| input | virtual pointer and keyboard protocols | RemoteDesktop → libei |
| clipboard | `ext-data-control-v1` | Mutter's selection API |

xrdp itself never opens anything in the user's runtime directory. On each connect, sesman's
session process does it and passes the open file descriptors to xrdp: the compositor's
Wayland socket for sway, and for GNOME the user's session bus and PipeWire socket, opened by
a child that has switched to the user. The module then joins xrdp's single event loop, with
no extra threads. End-to-end tests run in containers: 17/17 checks for sway,
26/26 for sway under a real systemd login, and 30/30 for GNOME Shell 50.5.

Around it, smaller pull requests make xrdp smoother on slow links: pacing the graphics
pipeline on frame acknowledgements ([#3912](https://github.com/neutrinolabs/xrdp/pull/3912),
[#3913](https://github.com/neutrinolabs/xrdp/pull/3913)), keeping the send backlog out of
the kernel with `TCP_NOTSENT_LOWAT` ([#3911](https://github.com/neutrinolabs/xrdp/pull/3911)),
and Opus sound and microphone
([#3914](https://github.com/neutrinolabs/xrdp/pull/3914),
[#3915](https://github.com/neutrinolabs/xrdp/pull/3915),
[#3922](https://github.com/neutrinolabs/xrdp/pull/3922)).

## 4. GNOME Remote Desktop

GNOME has its own RDP server, and six merge requests are in review there:
software H.264 with x264 ([!433](https://gitlab.gnome.org/GNOME/gnome-remote-desktop/-/merge_requests/433)),
AV1 encoding for FreeRDP's AV1 extension ([!434](https://gitlab.gnome.org/GNOME/gnome-remote-desktop/-/merge_requests/434)),
a fix for audio clicks ([!437](https://gitlab.gnome.org/GNOME/gnome-remote-desktop/-/merge_requests/437)),
clipboard clean-ups ([!439](https://gitlab.gnome.org/GNOME/gnome-remote-desktop/-/merge_requests/439),
[!440](https://gitlab.gnome.org/GNOME/gnome-remote-desktop/-/merge_requests/440)) and
client drive redirection ([!442](https://gitlab.gnome.org/GNOME/gnome-remote-desktop/-/merge_requests/442)).

## 5. Try it: rdp-release-kit

Upstream review takes time, and people want to try things now.
[rdp-release-kit](https://github.com/congzhangzh/rdp-release-kit) builds both ends from
pinned upstream sources (each with its SHA-256), and it changes no upstream code. Its targets:

- `sdl-freerdp` for Linux: one archive that runs on any x86_64 or aarch64 distribution with
  glibc 2.34 or newer, in X11 and Wayland sessions, with AV1 and Opus.
- The same client for Windows 10 and 11, on x86_64 and ARM64.
- The web client, built from FreeRDP master plus the open pull requests above.
- xrdp packages (.deb and .rpm) for Debian, Ubuntu and Fedora, and an xrdp preview with the
  Wayland work.

Published so far, as dated pre-releases: the web client and the xrdp preview. Stable
releases of the clients and packages come next.

The demo videos on the [FreeRDP Web page](https://congzhangzh.github.io/freerdp-web/) are
recorded by the kit's CI: GNOME and KDE Plasma on Wayland through the xrdp preview in AV1,
Xfce on xrdp, and GNOME Remote Desktop, all in a browser.

## How you can help

- **Try it** and open an issue when something breaks, with your browser, server and desktop.
- **Review or test** the open pull requests linked above. Upstream maintainers' time is the
  scarcest resource, and a "tested on X, works" comment really helps.
- The [xrdp Wayland RFC](https://github.com/neutrinolabs/xrdp/pull/3904) especially wants
  feedback on its design.

Next posts will go deeper into single pieces: how AV1 reaches the GPU through WebCodecs, and
how xrdp joins a Wayland session without giving up its privilege separation.
