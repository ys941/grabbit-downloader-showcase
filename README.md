<div align="center">

# 🐇 Grabbit

### Paste a link. Get the file. Instagram Reels, YouTube videos & Shorts — as **MP4** or **MP3**.

<p>
  <img src="https://img.shields.io/badge/Flask-3.0-000000?style=for-the-badge&logo=flask&logoColor=white" alt="Flask" />
  <img src="https://img.shields.io/badge/yt--dlp-extractor-FF0000?style=for-the-badge&logo=youtube&logoColor=white" alt="yt-dlp" />
  <img src="https://img.shields.io/badge/ffmpeg-remux%20%C2%B7%20transcode-007808?style=for-the-badge&logo=ffmpeg&logoColor=white" alt="ffmpeg" />
</p>
<p>
  <img src="https://img.shields.io/badge/Docker-deployable-2496ED?style=for-the-badge&logo=docker&logoColor=white" alt="Docker" />
  <img src="https://img.shields.io/badge/PWA-installable-5A0FC8?style=for-the-badge&logo=pwa&logoColor=white" alt="PWA" />
  <img src="https://img.shields.io/badge/status-live-2ea44f?style=for-the-badge" alt="Status: live" />
</p>

<em>Runs on a 512 MB free-tier box. Files are named after the video. Plays on an iPhone.</em>

</div>

> **About this repo.** This is a **public overview** of a private project. It showcases the architecture and the engineering behind it — the source code is kept private.

---

## What it is

**Grabbit** is a small, self-hosted web app that downloads public **Instagram Reels**, **YouTube videos** and **YouTube Shorts** as video (MP4) or audio (MP3 320 kbps / M4A). It began life as a Tkinter desktop script and was rebuilt as a deployable web service with a glassmorphism UI, an access-key lock screen, and a container image that survives a free tier.

Most of the interesting work in this project isn't the download — it's everything that goes wrong *after* the download.

---

## ✨ Highlights

### 🎯 The basics, done properly
- **Multiple links at once** — add and remove link rows; each link triggers its **own** browser save (no zip archive to unpack), with a live "downloading *n* of *m*" tally and per-file success/failure reporting.
- **Files are named after the video title**, Unicode preserved, illegal characters stripped — not `video_dQw4w9WgXcQ.mp4`.
- **Audio mode** defaults to MP3 320 kbps, with M4A available.
- **Access-key lock screen** — a full-screen gate on load, verified server-side, remembered locally and **re-verified on every load**, so rotating the key instantly re-locks every client. A mid-session rejection re-locks too. Deploy without a key and the gate disappears entirely.
- **Installable PWA** with a hand-drawn gradient icon set and a length cap that rejects oversized jobs before they start.

### 🍏 The iPhone bug that took a probe to find
The app reported HTTP 200 and delivered a file — that iPhones refused to play. Probing the **delivered bytes** (rather than trusting the status code) showed **VP9**, which no Apple device can decode. Two stacked causes:

1. The format selector ranked **resolution above codec**, so a VP9 rendition beat an H.264 one.
2. The real one: the video filter conflated the extractor's `"none"` (definitively *no* video) with a **missing** codec field (*unknown*). Instagram's progressive MP4 — the H.264 one — reports an unknown codec, so it was **discarded entirely**, leaving only the rendition that advertises VP9. A re-encode was therefore unavoidable on every single Reel.

Now candidates are ranked **known-H.264 → unknown → known-incompatible**, and the best compatibility tier is selected *before* resolution filtering. Result: Reels deliver **H.264 / yuv420p / AAC / faststart** by **stream copy**, no re-encode, in about 11 seconds. The trade-off is explicit and deliberate — 720p that plays beats 1080p that doesn't.

### 💥 The out-of-memory saga
A 512 MB instance kept getting OOM-killed — and `SIGKILL` leaves **no traceback**, so the logs were clean and the evidence was only in the platform's event feed. Two causes:

1. The finished file was parked in RAM as bytes, then copied *again* into a buffer at send time.
2. **ffmpeg.** Its memory bills the same container cgroup but is invisible in the Python process's RSS — a single 1080×1920 re-encode breached the limit on its own.

Fixes: stream straight off disk with a proper cleanup hook (the obvious callback never fires on pass-through responses — a real gotcha), a hard cap on concurrent jobs with a clean 429, single-threaded ffmpeg, a one-at-a-time encode semaphore, and stage-transition logs that print resident **and** cgroup memory so the last line before a kill names the phase. Verified in production: a 19 MB Reel peaks at **129 MB**.

### 🤖 Getting past "sign in to confirm you're not a bot"
Datacenter IPs get challenged where home connections don't. Grabbit works down a **client ladder** — it retries a blocked video across successive player clients automatically, and only for genuinely retryable failures; a private or removed video fails fast in under five seconds instead of grinding through every rung.

Where the platform demands an **account** rather than merely suspecting a bot, the honest answer is cookies: a credentials file is auto-detected from several conventional locations (including mounted secret files), with an explicit override and a boot log line stating which one was found. A proof-of-origin token provider is built into the image via a multi-stage build for the cases where it helps — and the README-worthy finding is documented rather than hidden: it does **not** rescue a login-required refusal, because the refusal happens before a token is ever requested.

### 🔋 Staying awake on a free tier
A self-ping loop starts at import (idempotent per process, so it survives worker restarts) and auto-detects its own public URL from the host's environment, going silent in local development. Interval and kill-switch are environment-configurable.

---

## 🧱 Tech stack

| Layer | Tech |
|---|---|
| Backend | **Flask 3** + **gunicorn** |
| Extraction | **yt-dlp** with per-attempt option building and a client ladder |
| Media | **ffmpeg** — remux by stream copy where possible, transcode only as fallback |
| Frontend | Single self-contained HTML page — glassmorphism UI, no build step, no framework |
| Packaging | **Docker** (multi-stage) · Nixpacks config · Procfile · health check endpoint |
| Hosting | Render / Railway blueprints included |

---

## 🗺️ How a job flows

```
   paste link(s)  ──▶  access-key gate  ──▶  per-link request
                                                    │
                                    ┌───────────────┴────────────────┐
                                    │  format selection              │
                                    │  compat tier ▸ then resolution │
                                    └───────────────┬────────────────┘
                                                    │
                              client ladder ◀───────┤ retryable failure?
                              (transparent retry)   │
                                                    ▼
                                   ffmpeg: stream copy ─▶ (fallback) re-encode
                                                    │      · single-threaded
                                                    │      · one at a time
                                                    ▼
                              per-job dir on disk ─▶ streamed to browser
                                                    ─▶ swept by a reaper
```

---

## 📌 Status

Deployed and live behind an access key, running on a free-tier container with the memory ceiling actively engineered around. This public README is a portfolio overview — the implementation is private.

Grabbit is a tool for downloading **content you have the right to download**. Respect the terms of service of the platforms you use it with.

---

## 📬 Contact the developer

Interested in this or **any of Bhardwaj's projects** — a demo, a walkthrough, collaboration, or licensing?

### 📧 **[ys9410017064@gmail.com](mailto:ys9410017064@gmail.com)**

---

<div align="center">

<img src="assets/heart.gif" width="26" height="26" alt="beating heart" />

**Made with love by Yati Bhardwaj**

<sub><a href="https://github.com/ys941">github.com/ys941</a> · <a href="mailto:ys9410017064@gmail.com">ys9410017064@gmail.com</a></sub>

</div>
