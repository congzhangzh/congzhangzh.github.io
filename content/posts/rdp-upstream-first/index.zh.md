---
title: "上游优先的 Linux 远程桌面：AV1、Wayland 与浏览器客户端"
date: 2026-10-22
summary: "我们在 FreeRDP、xrdp 和 GNOME Remote Desktop 上做的事：从服务器到浏览器的 AV1、xrdp 的 Wayland 会话，以及今天就能试用的可复现构建。所有改动都先提交上游。"
tags: ["rdp", "freerdp", "xrdp", "gnome-remote-desktop", "av1", "wayland", "webassembly"]
---

说到 Linux 远程桌面，很多人的印象还停留在 X11 会话、老旧的编码，外加一个必须先装好的客户端。
我们想要的是另一种样子：一套开源方案，**快**（端到端 AV1），支持 **Wayland** 桌面
（GNOME、KDE Plasma、sway），而且**打开浏览器标签页就能用**，什么都不用装。

这些工作都不放在私有分支里。每个改动都以 Pull Request（FreeRDP、xrdp）或 Merge Request
（GNOME Remote Desktop）的形式提交给上游项目；我们发布的构建也只使用上游源码。
这篇文章是一张总图：已经有什么、哪些已合入、哪些还在评审。文中数字截至 2026 年 10 月 22 日。

## 整体结构

```
浏览器标签页：FreeRDP SDL3 客户端（编译为 WebAssembly）
  ├─ AV1 / H.264 → WebCodecs
  └─ 声音、麦克风 → Web Audio
        ▲
        │  RDP
        ▼
Linux 服务器：xrdp ── wlup ── GNOME / KDE / sway（由 sesman 启动 Wayland 会话）
  └─ 或 gnome-remote-desktop（GNOME 自带的 RDP 服务器）

两端的可复现构建：rdp-release-kit
```

| 项目 | 我们提交的 PR / MR | 状态 |
|---|---|---|
| [FreeRDP](https://github.com/FreeRDP/FreeRDP/pulls?q=author%3Acongzhangzh) | 2026 年 9 月 30 日以来 45 个 | 已合入 33 个，评审中 6 个 |
| [xrdp](https://github.com/neutrinolabs/xrdp/pulls?q=author%3Acongzhangzh) | 9 个 | 已合入 1 个，评审中 8 个（含 Wayland RFC） |
| [pulseaudio-module-xrdp](https://github.com/neutrinolabs/pulseaudio-module-xrdp/pull/143) | 1 个 | 评审中 |
| [GNOME Remote Desktop](https://gitlab.gnome.org/GNOME/gnome-remote-desktop/-/merge_requests?author_username=congzhangzh) | 6 个 | 评审中 |

## 一、浏览器里的 FreeRDP

[FreeRDP Web](https://congzhangzh.github.io/freerdp-web/) 运行的是 FreeRDP 自己的 SDL3
客户端，用 Emscripten 编译成 WebAssembly。它不是另起炉灶、用 JavaScript 重写一遍 RDP，
而是和桌面客户端同一份 C 代码，所以每一个协议层的修复，两边同时受益。

关键在于让浏览器替我们干活。在 WebAssembly 里软解 AV1 很吃 CPU，可浏览器本身就带有视频解码器，
往往还有硬件加速，并通过 **WebCodecs** API 开放出来。于是客户端把每一帧压缩数据交给
WebCodecs，由浏览器解码，能用 GPU 时就用 GPU。（目前已在 Chrome 上测试。）

上游进展：

- 已合入：FreeRDP CI 中的 Emscripten 构建与单元测试任务
  （[#13728](https://github.com/FreeRDP/FreeRDP/pull/13728)）、为此所需的构建修复
  （[#13724](https://github.com/FreeRDP/FreeRDP/pull/13724)）、
  **用 WebCodecs 解码 AV1**（[#13731](https://github.com/FreeRDP/FreeRDP/pull/13731)）、
  **通过 Web Audio 播放声音**（[#13732](https://github.com/FreeRDP/FreeRDP/pull/13732)），
  以及 Opus 麦克风编码修复（[#13810](https://github.com/FreeRDP/FreeRDP/pull/13810)）。
- 评审中：浏览器剪贴板（[#13733](https://github.com/FreeRDP/FreeRDP/pull/13733)）、
  Web Audio 麦克风（[#13811](https://github.com/FreeRDP/FreeRDP/pull/13811)），
  以及通过 WebCodecs 解码 H.264（[#13812](https://github.com/FreeRDP/FreeRDP/pull/13812)）。

## 二、桌面端的 FreeRDP

做浏览器客户端的过程中，不断碰到所有 FreeRDP 用户都会遇到的问题，所以很多已合入的改动其实是给
Linux、Windows 和 macOS 上的原生客户端的：

- **AV1**：修正 4:4:4 帧的颜色（[#13561](https://github.com/FreeRDP/FreeRDP/pull/13561)），
  以及 pkg-config 找不到时也能链接 dav1d 和 aom
  （[#13625](https://github.com/FreeRDP/FreeRDP/pull/13625)）。
- **声音**：PulseAudio、Windows、macOS 上播放更平稳
  （[#13722](https://github.com/FreeRDP/FreeRDP/pull/13722)、
  [#13723](https://github.com/FreeRDP/FreeRDP/pull/13723)、
  [#13727](https://github.com/FreeRDP/FreeRDP/pull/13727)、
  [#13574](https://github.com/FreeRDP/FreeRDP/pull/13574)）。
- **剪贴板与文件**：Windows 上的图片与 UTF-8 文本
  （[#13633](https://github.com/FreeRDP/FreeRDP/pull/13633)）、与 macOS 访达之间复制粘贴文件
  （[#13591](https://github.com/FreeRDP/FreeRDP/pull/13591)），以及连接基于 FreeRDP 的服务器时的驱动器重定向
  （[#13592](https://github.com/FreeRDP/FreeRDP/pull/13592)、
  [#13593](https://github.com/FreeRDP/FreeRDP/pull/13593)）。
- **调整大小与多显示器**：重新激活时重置编码器
  （[#13668](https://github.com/FreeRDP/FreeRDP/pull/13668)）、向新的显示通道重发布局
  （[#13726](https://github.com/FreeRDP/FreeRDP/pull/13726)），以及区分桌面缩放与设备缩放
  （[#13806](https://github.com/FreeRDP/FreeRDP/pull/13806)）。

## 三、xrdp 支持 Wayland

xrdp 是 Linux 上最常用的 RDP 服务器，但目前只服务 X11 会话。RFC
[neutrinolabs/xrdp#3904](https://github.com/neutrinolabs/xrdp/pull/3904) 新增了一个后端模块
**wlup**，以及两种由 sesman 像其他会话一样启动的新会话类型：

| | sway（wlroots） | GNOME（Mutter） |
|---|---|---|
| 画面 | `ext-image-copy-capture-v1`，只传变化区域 | ScreenCast → PipeWire |
| 输入 | 虚拟指针与虚拟键盘协议 | RemoteDesktop → libei |
| 剪贴板 | `ext-data-control-v1` | Mutter 的 selection API |

xrdp 本身从不打开用户运行时目录里的任何东西。每次连接时，由 sesman 的会话进程负责打开，再把已打开的文件描述符传给
xrdp：sway 传的是合成器的 Wayland 套接字；GNOME 则由一个已切换到该用户身份的子进程打开用户的会话总线和 PipeWire
套接字。之后模块接入 xrdp 的单线程事件循环，不额外开线程。端到端测试在容器中运行：sway 17/17，
真实 systemd 登录下的 sway 26/26，GNOME Shell 50.5 30/30。

此外还有几个小 PR，让 xrdp 在慢速网络上更顺畅：按帧确认节奏推送图形
（[#3912](https://github.com/neutrinolabs/xrdp/pull/3912)、
[#3913](https://github.com/neutrinolabs/xrdp/pull/3913)）、用 `TCP_NOTSENT_LOWAT`
避免发送积压堆在内核里（[#3911](https://github.com/neutrinolabs/xrdp/pull/3911)），以及 Opus 声音与麦克风
（[#3914](https://github.com/neutrinolabs/xrdp/pull/3914)、
[#3915](https://github.com/neutrinolabs/xrdp/pull/3915)、
[#3922](https://github.com/neutrinolabs/xrdp/pull/3922)）。

## 四、GNOME Remote Desktop

GNOME 有自己的 RDP 服务器，那边有六个 MR 正在评审：基于 x264 的软件 H.264 编码
（[!433](https://gitlab.gnome.org/GNOME/gnome-remote-desktop/-/merge_requests/433)）、
面向 FreeRDP AV1 扩展的 AV1 编码
（[!434](https://gitlab.gnome.org/GNOME/gnome-remote-desktop/-/merge_requests/434)）、
修复音频爆音（[!437](https://gitlab.gnome.org/GNOME/gnome-remote-desktop/-/merge_requests/437)）、
剪贴板相关清理（[!439](https://gitlab.gnome.org/GNOME/gnome-remote-desktop/-/merge_requests/439)、
[!440](https://gitlab.gnome.org/GNOME/gnome-remote-desktop/-/merge_requests/440)），
以及客户端驱动器重定向（[!442](https://gitlab.gnome.org/GNOME/gnome-remote-desktop/-/merge_requests/442)）。

## 五、动手试试：rdp-release-kit

上游评审需要时间，而大家现在就想试。
[rdp-release-kit](https://github.com/congzhangzh/rdp-release-kit) 用锁定版本的上游源码（每个都带
SHA-256）构建两端，不改动任何上游代码。它的目标产物：

- Linux 版 `sdl-freerdp`：一个压缩包，可在任何 glibc 2.34 及以上的 x86_64 或 aarch64 发行版上运行，
  X11 和 Wayland 会话都能用，带 AV1 和 Opus。
- 同一个客户端的 Windows 10/11 版本，支持 x86_64 和 ARM64。
- Web 客户端，基于 FreeRDP master 加上前面那些还在评审的 PR。
- Debian、Ubuntu、Fedora 的 xrdp 软件包（.deb / .rpm），以及包含 Wayland 工作的 xrdp 预览版。

目前已发布的是按日期命名的预发布版：Web 客户端和 xrdp 预览版。客户端和软件包的正式版是下一步。

[FreeRDP Web 页面](https://congzhangzh.github.io/freerdp-web/) 上的演示视频由该工具链的 CI 录制：
经 xrdp 预览版以 AV1 传输的 Wayland 版 GNOME 和 KDE Plasma、xrdp 上的 Xfce，以及 GNOME Remote Desktop，
全部在浏览器中运行。

## 欢迎参与

- **试用**，遇到问题就开 issue，并写明你的浏览器、服务器和桌面环境。
- **评审或测试**上面那些还开着的 PR。上游维护者的时间是最稀缺的资源，一句「在 X 上测过，可用」就很有帮助。
- 特别是 [xrdp 的 Wayland RFC](https://github.com/neutrinolabs/xrdp/pull/3904)，非常需要对其设计的反馈。

后续文章会逐个深入：AV1 如何经 WebCodecs 交给 GPU 解码，以及 xrdp 如何在不放弃权限分离的前提下接入 Wayland 会话。
