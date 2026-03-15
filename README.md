# Telegram Media Downloader v2

> **A powerful Python tool (Jupyter Notebook) that bulk-downloads all media from your Telegram channels and groups — including restricted/protected content — and optionally uploads them to your Saved Messages.**

---

## Disclaimer

> **This project is provided strictly for educational and research purposes only.**
>
> This tool demonstrates how Telegram's MTProto API works, how async Python can manage concurrent downloads/uploads, and how to build resilient CLI applications with retry logic and progress tracking.
>
> **The authors and contributors are NOT responsible for how this tool is used.** Users are solely responsible for ensuring their use complies with:
> - Telegram's [Terms of Service](https://telegram.org/tos)
> - Applicable copyright laws in their jurisdiction
> - The intellectual property rights of content creators
>
> Downloading or redistributing copyrighted content without authorization may violate local laws. **Use at your own risk.**
>
> This tool does NOT bypass encryption, crack passwords, or exploit vulnerabilities. It uses Telegram's official MTProto API with your own authenticated account — the same API that every Telegram client (desktop, mobile, web) uses.

---

## Table of Contents

- [Introduction](#introduction)
- [Features](#features)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Getting Your Telegram API Credentials](#getting-your-telegram-api-credentials)
  - [Installation](#installation)
- [Functionality Flow](#functionality-flow)
  - [Download Flow](#download-flow)
  - [Upload Flow (Send to Saved Messages)](#upload-flow-send-to-saved-messages)
  - [How Restricted Channel Downloads Work](#how-restricted-channel-downloads-work)
- [Walkthrough: A Complete Example Session](#walkthrough-a-complete-example-session)
  - [Step 1: Launch & Authentication](#step-1-launch--authentication)
  - [Step 2: Channel Selection](#step-2-channel-selection)
  - [Step 3: Media Scanning & File Selection](#step-3-media-scanning--file-selection)
  - [Step 4: Downloading](#step-4-downloading)
  - [Step 5: Upload to Saved Messages](#step-5-upload-to-saved-messages)
  - [Step 6: Session Summary](#step-6-session-summary)
- [Main Menu Reference](#main-menu-reference)
- [File Structure](#file-structure)
- [Configuration](#configuration)
  - [Tunable Constants](#tunable-constants)
  - [Custom File Naming](#custom-file-naming)
- [Technical Deep Dive](#technical-deep-dive)
  - [Upload Speed and FloodPremiumWait](#upload-speed-and-floodpremiumwait)
  - [Upload Pipeline Patch (Queue(1) to Queue(16))](#upload-pipeline-patch-queue1-to-queue16)
  - [SQLite Double-Patch Guard](#sqlite-double-patch-guard)
  - [Error Handling Matrix](#error-handling-matrix)
  - [Tracker Formats](#tracker-formats)
- [Known Issues & Warnings](#known-issues--warnings)
  - [Issue 1: FLOOD_PREMIUM_WAIT Log Spam During Uploads](#issue-1-flood_premium_wait-log-spam-during-uploads)
  - [Issue 2: Free Account Upload Speed Cap](#issue-2-free-account-upload-speed-cap)
  - [Issue 3: Pyrogram WARNING Logs During Throttle](#issue-3-pyrogram-warning-logs-during-throttle)
  - [Other Known Limitations](#other-known-limitations)
- [Dependencies](#dependencies)
- [Future Work](#future-work)
- [Troubleshooting](#troubleshooting)
- [Security Notes](#security-notes)
- [License & Disclaimer](#license--disclaimer)

---

## Introduction

**Telegram Media Downloader v2** is an asynchronous Python application packaged as a **Jupyter Notebook** (`telegram_downloader.ipynb` — 76 cells, ~2,136 lines of code) designed to run on **Google Colab** or any Jupyter environment. It lets you bulk-download every type of media from your Telegram channels and groups — photos, videos, documents, audio files, voice messages, video notes, GIFs, and stickers — **including content from restricted/protected channels** where Telegram's official apps disable the "Save" and "Forward" buttons.

### The Problem It Solves

Telegram allows channel admins to enable **"Restrict saving content"**, which disables forwarding, saving, and screenshotting in official Telegram clients. However, this is a **client-side UI restriction**, not an API-level one. If you are a legitimate member of a channel, Telegram's MTProto API still serves you the full media data — the official apps simply choose not to expose the save/forward buttons.

This tool uses **PyroFork** (a maintained fork of Pyrogram) to authenticate as your own Telegram user account via MTProto, giving you direct API access to download media you are authorized to view.

### What Makes This Tool Different

Unlike simple download scripts, this tool was built for **real-world usage on large channels** with hundreds of files and multi-gigabyte videos:

- **Chunked resume** — Interrupted 2 GB downloads resume from where they left off, not from zero
- **Intelligent throttle handling** — Telegram's `FloodWait` and `FloodPremiumWait` rate limits are automatically detected, waited out, and retried
- **Zero data loss** — `download_tracker.json` and `sent_tracker.json` ensure you never re-download or re-upload a file, even across sessions or Colab crashes
- **Optimized for Google Colab** — Auto-installs dependencies, patches the event loop, enables `uvloop`, and runs at 5-10 MB/s download speed (vs ~200 KB/s locally on free accounts)
- **Upload to Saved Messages** — After downloading, you can send files to your Telegram Saved Messages with parallel uploads, photo batching, and progress tracking

---

## Features

### Core Download Engine
- **Restricted content download** — Downloads from channels with "Restrict saving content" enabled
- **Media breakdown by type** — Scans each channel and shows: photos, videos, documents, audio, voice notes, video notes, GIFs with estimated sizes
- **Per-file selection** — Videos/documents get individual file listings with date, size, and filename columns (sorted newest first); photos/audio/voice get batch yes/no
- **Pre-download Saved Messages scan (opt-in)** — Scans Saved Messages before downloading to skip files you've already saved
- **Chunked resume** — Files >50 MB use `stream_media()` with 1 MB chunks. Interrupted downloads resume from where they left off via `.part` files
- **FileReferenceExpired retry** — Automatically re-fetches message when Telegram's file token expires during long downloads (3 attempts)
- **FloodWait + FloodPremiumWait handling** — Automatically sleeps and retries when Telegram throttles requests
- **Download tracker** — `download_tracker.json` tracks every downloaded message ID per channel. Re-runs skip already-downloaded files
- **O(1) duplicate detection** — Cached set-based lookups instead of scanning lists
- **0-byte file cleanup** — Detects and removes broken/empty downloads automatically

### Send to Saved Messages (Upload Engine)
- **Upload downloaded files to Telegram Saved Messages** — Select channels and files to send
- **Upload mode selection** — Choose between Documents (fast, no media processing) or Media (with previews)
- **Parallel uploads** — Up to 3 simultaneous file uploads via `asyncio.Semaphore`
- **Photo batching** — Groups up to 10 small photos (<10 MB each) per `send_media_group()` API call
- **Capped FloodWait retries** — Max 10 flood retries per file before skipping (prevents infinite retry loops)
- **Smart remaining-file detection** — Shows already-sent count vs remaining; numbered menu: `[1] Send remaining N`, `[2] Send all`, `[3] Select`, `[4] Skip`
- **Saved Messages scan (opt-in)** — Scans Saved Messages captions to recover sent-tracker state after Colab session loss
- **Sent tracker** — `sent_tracker.json` prevents re-uploading files already sent
- **Adaptive upload delays** — No delay for small files; 1s (50-100MB), 10s (100-500MB), 20s (>500MB) for large files only
- **Upload progress with speed/ETA** — Real-time progress bar for files >50 MB
- **Premium detection** — Auto-detects Premium accounts (4 GB upload limit vs 2 GB free)
- **Ctrl+C safe** — Saves sent tracker on interrupt during upload

### Performance Optimizations
- **Upload pipeline patch** — Monkey-patches Pyrogram's `save_file()` upload queue from `Queue(1)` to `Queue(16)` — **3-8x upload speed improvement**
- **`sleep_threshold=120`** — Lets Pyrogram handle FloodWait ≤120s internally so upload workers retry chunks instead of skipping them
- **uvloop on Colab** — 2-4x faster async I/O event loop (Linux only, auto-installed)
- **TgCrypto** — C-based AES encryption for faster MTProto operations
- **3 concurrent download streams** — `max_concurrent_transmissions=3`

### User Experience
- **Main menu loop** — Persistent session with 7 options, no need to restart between operations
- **Channel stats dashboard** — Per-channel download counts, last updated dates, total disk usage
- **Session stats** — Live counters: files downloaded, failed, total size, elapsed time
- **Retry failed downloads** — One-click retry of all failed files with fresh file references
- **Graceful Ctrl+C** — Saves tracker + adds remaining queue to retry list on interrupt
- **Custom file naming** — Template system with `{filename}`, `{date}`, `{msgid}`, `{type}`, `{ext}` placeholders
- **Preserve file dates** — Sets file modification time to original Telegram message date
- **Estimated sizes per type** — Samples up to 50 files per media type with stride-based spread
- **Batch progress** — Real-time speed and ETA after each file

### Platform Support
- **Windows** — UTF-8 output wrapping, `WindowsSelectorEventLoopPolicy`
- **macOS / Linux** — Standard asyncio
- **Google Colab** — Auto-installs deps, `nest_asyncio` patch, uvloop, downloads to `/content/telegram_downloader/`

---

## Getting Started

### Prerequisites

1. **Python 3.8+** (Python 3.10+ recommended)
2. **A Telegram account** — The tool logs in as your user account (not a bot)
3. **Telegram API credentials** — `api_id` and `api_hash` (free, see below)

### Getting Your Telegram API Credentials

This is a **one-time setup**. Telegram requires all third-party apps to register for API access. This is free and takes about 2 minutes.

#### Step-by-step:

1. **Open your browser** and go to [https://my.telegram.org](https://my.telegram.org)

2. **Log in with your phone number**
   - Enter your phone number in international format (e.g., `+1234567890`)
   - Telegram will send a confirmation code **to your Telegram app** (not SMS)
   - Enter the code on the website

3. **Go to "API Development Tools"**
   - After logging in, you'll see a page with links
   - Click **"API development tools"**

4. **Create a new application**
   - If you haven't created an app before, you'll see a form:
     - **App title**: Anything (e.g., "My Downloader")
     - **Short name**: Anything (e.g., "mydownloader")
     - **Platform**: Can leave as default
     - **Description**: Optional
   - Click **"Create application"**

5. **Note your credentials**
   - You'll see a page with your app details
   - Copy these two values:
     - **`api_id`** — A number (e.g., `12345678`)
     - **`api_hash`** — A string (e.g., `0123456789abcdef0123456789abcdef`)
   - **Keep these private** — they're tied to your Telegram account

> **Important:** These credentials are NOT a bot token. They identify your application to Telegram's API. Every Telegram client (desktop, mobile, web) uses similar credentials.

### Installation

#### Option A: Google Colab (Recommended for speed)

1. Open [Google Colab](https://colab.research.google.com/)
2. Upload `telegram_downloader.ipynb` (or open it directly from GitHub/Google Drive)
3. Click **Runtime → Run all** (or run cells one by one)
4. Dependencies auto-install in the first cell

**Typical Colab download speeds:** ~5-10 MB/s (vs ~200 KB/s locally on free accounts)

#### Option B: Local Jupyter (Windows/macOS/Linux)

```bash
# Install dependencies
pip install pyrofork TgCrypto-pyrofork jupyter

# Launch Jupyter and open the notebook
jupyter notebook telegram_downloader.ipynb
```

Then run all cells in order.

#### First Run

1. Enter your **API ID** and **API Hash** when prompted (saved to `tg_config.json` for future runs)
2. Enter your **phone number** in international format (e.g., `+1234567890`)
3. Enter the **OTP code** sent to your Telegram app
4. You're logged in — your session is saved to `my_telegram_session.session` (no need to re-authenticate on future runs)

---

## Functionality Flow

### Download Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    DOWNLOAD FLOW                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Authenticate via MTProto (phone + OTP, session cached)  │
│                         │                                   │
│  2. Fetch channel list via get_dialogs()                    │
│                         │                                   │
│  3. User selects channel(s) to download from                │
│                         │                                   │
│  4. Scan media counts per type (server-side, fast)          │
│     └── search_messages_count() for each filter             │
│                         │                                   │
│  5. Estimate sizes by sampling up to 50 files per type      │
│     └── Stride-based spread across channel timeline         │
│                         │                                   │
│  6. User selects media types (photos, videos, docs, etc.)   │
│                         │                                   │
│  7. [Optional] Scan Saved Messages to skip already-saved    │
│                         │                                   │
│  8. File listing:                                           │
│     ├── Videos/Documents: individual listing with date,     │
│     │   size, name (sorted newest first)                    │
│     └── Photos/Audio/Voice: batch yes/no prompt             │
│                         │                                   │
│  9. User selects specific files or confirms batch           │
│                         │                                   │
│  10. Download:                                              │
│      ├── < 50 MB: download_media() (standard Pyrogram)      │
│      └── ≥ 50 MB: stream_media() (chunked, resumable)      │
│                         │                                   │
│  11. Track downloaded IDs in download_tracker.json           │
│  12. Preserve file dates via os.utime()                     │
│  13. Clean up 0-byte files from failed downloads            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Upload Flow (Send to Saved Messages)

```
┌─────────────────────────────────────────────────────────────┐
│              UPLOAD TO SAVED MESSAGES FLOW                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Scan downloaded channel folders on disk                 │
│                         │                                   │
│  2. User selects channels to upload                         │
│                         │                                   │
│  3. [Optional] Scan Saved Messages to recover tracker       │
│     └── Matches by caption format: "ChannelName/filename"   │
│                         │                                   │
│  4. Upload mode selection:                                  │
│     ├── [1] Documents (fast — no media processing)          │
│     └── [2] Media (with previews — slower for videos)       │
│                         │                                   │
│  5. Per-channel smart menu:                                 │
│     ├── [1] Send remaining N files                          │
│     ├── [2] Send all files                                  │
│     ├── [3] Select individual files                         │
│     └── [4] Skip this channel                               │
│                         │                                   │
│  6. Filter out: 0-byte files, files > upload limit          │
│     └── 2 GB (free) or 4 GB (Premium)                       │
│                         │                                   │
│  7. Upload with:                                            │
│     ├── Up to 3 parallel uploads (asyncio.Semaphore)        │
│     ├── Photo batching (up to 10 per send_media_group)      │
│     ├── Adaptive delays (no delay for small files)          │
│     └── Real-time progress with speed and ETA               │
│                         │                                   │
│  8. Track sent files in sent_tracker.json                   │
│     └── Periodic saves every 10 files + on Ctrl+C          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### How Restricted Channel Downloads Work

Telegram's **"Restrict saving content"** is a **client-side UI flag**, not an API-level restriction.

```
┌──────────────────────────────────────────────────────────────┐
│  Official Telegram App                                       │
│  ┌──────────────────┐     ┌─────────────┐                   │
│  │ Channel message   │────>│ MTProto API  │  ← data flows    │
│  │ [Save disabled]   │     │ (full data)  │    normally       │
│  │ [Forward disabled]│     └──────┬──────┘                   │
│  └──────────────────┘            │                           │
│        The app HIDES         The API SERVES                  │
│        save/forward          all media data                  │
│        buttons               to authorized users             │
│                                                              │
│  PyroFork (this tool)                                        │
│  ┌──────────────────┐     ┌─────────────┐                   │
│  │ Authenticated as  │────>│ MTProto API  │  ← same API      │
│  │ your user account │     │ (full data)  │    same data      │
│  │ via MTProto       │     └──────┬──────┘                   │
│  └──────────────────┘            │                           │
│        Accesses the same     Downloads the                   │
│        API directly          full media files                │
└──────────────────────────────────────────────────────────────┘
```

You must be a **legitimate member** of the channel. This tool does not bypass any authentication or encryption — it simply uses the API directly instead of going through the official app's UI restrictions.

---

## Walkthrough: A Complete Example Session

Below is a realistic walkthrough based on an actual session log. This demonstrates the full download + upload flow on Google Colab.

### Step 1: Launch & Authentication

When you first run the tool, it displays the welcome banner and prompts for your API credentials:

```
  +======================================================+
  |         Telegram Media Downloader v2                 |
  |     Download media from your channels/groups         |
  |     (including restricted/protected content)         |
  +======================================================+
  | ENV: Google Colab (high-speed download)              |
  | SAVE TO: /content/telegram_downloader/               |
  +======================================================+

  First-time setup -- enter your Telegram API credentials.
  (Get them from https://my.telegram.org)

  Enter API ID: 12345678
  Enter API Hash: your_api_hash_here

  Save credentials for future runs? (y/n): y
  [OK] Credentials saved to /content/telegram_downloader/tg_config.json

  Connecting to Telegram...
  Enter phone number or bot token: +1234567890
  Is "+1234567890" correct? (y/N): y
  The confirmation code has been sent via Telegram app
  Enter confirmation code: 12345
  [OK] Logged in as: Your Name (@username) [Free]
```

> On subsequent runs, credentials and session are loaded automatically — no re-authentication needed.

### Step 2: Channel Selection

The main menu appears, and you choose option 1 to select channels:

```
  ============================================================
  MAIN MENU
  ============================================================
    1) Select channels to download
    2) Refresh channel list
    3) Show session stats
    4) Send files to Saved Messages
    5) Retry failed downloads
    6) Channel stats dashboard
    7) Exit
  ============================================================

  Choose option (1-7): 1
```

Your channels and groups are listed:

```
  Found 9 channels/groups:

     #  Type         Name                                      Members  Protected
  ----  ------------ ---------------------------------------- --------  ----------
     1  Channel      Public Channel                             15283
     2  Channel      Private Group                                549  [LOCKED]
     3  Channel      Premium Content                              250  [LOCKED]
     ...

  Select chats to process (e.g., 1,3,5 or 1-10 or all): 3

  Selected 1 chat(s):
    - Premium Content [PROTECTED]
```

The `[LOCKED]` tag indicates channels with "Restrict saving content" enabled. The tool downloads from these normally via the API.

### Step 3: Media Scanning & File Selection

The tool scans the channel's media and shows a breakdown:

```
  ============================================================
  Channel: Premium Content
  [PROTECTED CONTENT -- will download via API]
  ============================================================

  Scanning media in this channel...

  Media breakdown (96 total files):
  Estimating sizes...

    1) photos             50 files   ~21.35 MB     [batch selection]
    2) videos             40 files   ~14.87 GB     [individual selection]
    3) gifs                6 files   ~8.11 MB      [batch selection]

    Total estimated size: ~14.89 GB

    A) All media (96 files)
    S) Skip this channel

  Select media types (e.g., 1,2 or 1-3 or A for all, S to skip): 2
```

For videos/documents, you get individual file listings:

```
  --- VIDEOS (40 files) ---
  Fetching file list...

  40 videos available (14.87 GB, ~50m 44s):

     #        Date        Size  Filename
  ----  ----------  ----------  --------------------------------------------------
     1  09-03-2026    24.04 MB  video_129.mp4
     2  06-03-2026   208.91 MB  video_128.mp4
     3  06-03-2026     1.93 GB  video_127.mp4
     4  04-03-2026    91.09 MB  video_125.mp4
     ...

  Total: 14.87 GB | Estimated: ~50m 44s

  Select videos (e.g., 1 or 1-3 or 1,2 or all or skip): 1-4
  Queued 4 videos (2.25 GB, ~7m 40s)
```

### Step 4: Downloading

After confirmation, downloads begin with real-time progress:

```
  ============================================================
  DOWNLOAD QUEUE for: Premium Content
  ============================================================
    video               4 files
    Size:           2.25 GB
    Estimated time: ~7m 40s
  ============================================================

  Start downloading? (y/n): y

  [1/4] video_129.mp4
  >> video_129.mp4 | 100.0% | 24.04 MB/24.04 MB | 2.83 MB/s
  [pause 3s before next file...]

  [2/4] video_128.mp4
  [LARGE 208.91 MB] video_128.mp4
  >> video_128.mp4 | 100.0% | 208.91 MB/208.91 MB | 3.81 MB/s

  -- Batch: 2/4 done | 232.95 MB/2.25 GB | 3.51 MB/s | ETA: ~9m 49s --
  [pause 5s before next file...]

  [3/4] video_127.mp4
  [LARGE 1.93 GB] video_127.mp4
  >> video_127.mp4 | 100.0% | 1.93 GB/1.93 GB | 4.04 MB/s

  [4/4] video_125.mp4
  [LARGE 91.09 MB] video_125.mp4
  >> video_125.mp4 | 100.0% | 91.09 MB/91.09 MB | 3.56 MB/s

  -- Batch: 4/4 done | 2.25 GB/2.25 GB | 3.86 MB/s | ETA: ~0s --

  -- Summary for: Premium Content
     Downloaded: 4 files (2.25 GB)
     Failed: 0

  ============================================================
  DOWNLOAD BATCH COMPLETE
  ============================================================
     Chats processed:  1
     Session total:    4 downloaded, 0 failed
     Session size:     2.25 GB
     Session time:     10m 30s
  ============================================================
```

**Key observations:**
- Small files (<50 MB) download with standard progress
- Large files (≥50 MB) are tagged `[LARGE]` and use chunked downloading with resume support
- Adaptive delays between files (3s, 5s, 10s) reduce Telegram throttling
- Batch progress shows running speed average and ETA

### Step 5: Upload to Saved Messages

From the main menu, choose option 4 to upload downloaded files to your Saved Messages:

```
  Choose option (1-7): 4

  Downloaded channels (1):

     #   Files        Size  Channel Name
  ----  ------  ----------  ----------------------------------------
     1       4     2.25 GB  Premium Content

  Select channels to send (e.g., 1,2 or all or back): 1
  Scan Saved Messages to detect already-sent files? (y/n, default=n): y

  Scanning Saved Messages to detect already-uploaded files...
    Scanning: Premium Content... found 36 (36 recovered)
  Recovered 36 file(s) into sent tracker.
```

The Saved Messages scan recovers state from a previous session (useful after Colab crashes). Then you select upload mode:

```
  Upload mode:
    [1] Documents (faster — no media processing)
    [2] Media (with previews — slower for videos)
  Choose (1/2, default=1): 1
  Mode: Documents (fast)

  --- Premium Content (4 files, 2.25 GB) ---
  Already sent: 36, Remaining: 4
  Options: [1] Send remaining 4  [2] Send all 4  [3] Select  [4] Skip
  Choose (1/2/3/4): 1

  Uploading 4 files (2.25 GB) to Saved Messages...
  Mode: Documents | Parallel: up to 3 | Batch photos: up to 10

  Sending video_128.mp4 (208.91 MB)...
  Sending video_129.mp4 (24.04 MB)...
  Sending video_127.mp4 (1.93 GB)...
```

Uploads run in parallel (up to 3 concurrent). For free accounts, you'll see `FLOOD_PREMIUM_WAIT` messages in the log — this is normal (see [Known Issues](#known-issues--warnings)). The uploads complete successfully despite the throttling:

```
  [OK] Sent in 4m 30s       ← video_129.mp4 (24 MB)
  [OK] Sent in 15m 8s       ← video_128.mp4 (209 MB)
  Sending video_125.mp4 (91.09 MB)...
  [OK] Sent in 27m 3s       ← video_127.mp4 (1.93 GB)

  Premium Content: Sent 4, Failed 0 | 2.25 GB in 27m 23s (1.40 MB/s)

  [OK] Total sent to Saved Messages: 4, Failed: 0
```

### Step 6: Session Summary

When you exit (option 7), you get a final session summary:

```
  ============================================================
  SESSION SUMMARY
  ============================================================
     Chats processed:  1
     Files downloaded: 4
     Files failed:     0
     Total size:       2.25 GB
     Time taken:       45m 4s
     Output directory: /content/telegram_downloader/telegram_downloads
  ============================================================

  Disconnected from Telegram.
```

**Total time breakdown for this session:**
- Download: ~10 minutes (4 files, 2.25 GB at ~3.8 MB/s)
- Upload: ~27 minutes (4 files, 2.25 GB at ~1.4 MB/s — throttled by Telegram)
- Menu/scanning/delays: ~8 minutes

---

## Main Menu Reference

```
  ============================================================
  MAIN MENU
  ============================================================
    1) Select channels to download
    2) Refresh channel list
    3) Show session stats
    4) Send files to Saved Messages
    5) Retry failed downloads (3 failed)
    6) Channel stats dashboard
    7) Exit
  ============================================================
```

| Option | Description |
|--------|-------------|
| **1** | Browse your channels, pick media types, select individual files, and download them |
| **2** | Re-fetch your channel/group list from Telegram (if you joined new ones since starting) |
| **3** | Show current session counters: files downloaded, failed, total size, elapsed time |
| **4** | Upload previously downloaded files to your Telegram Saved Messages |
| **5** | Retry all files that failed in this session (re-fetches fresh file references from Telegram) |
| **6** | Dashboard: per-channel file counts, last updated dates, total disk usage |
| **7** | Save all tracker data to disk and disconnect from Telegram |

---

## File Structure

```
telegram_downloader/
  telegram_downloader.ipynb     # Main notebook (76 cells, ~2,136 lines of code)
  requirements.txt              # pyrofork, TgCrypto-pyrofork
  README.md                     # This documentation file
  tg_config.json                # API credentials (auto-created on first run)
  my_telegram_session.session   # Pyrogram session file (DO NOT SHARE)
  download_tracker.json         # Tracks downloaded message IDs per channel
  sent_tracker.json             # Tracks files already sent to Saved Messages
  telegram_downloads/           # Downloaded media, organized by channel name
    ChannelName1/
      video_123.mp4
      photo_456.jpg
    ChannelName2/
      document_789.pdf
```

---

## Configuration

### Tunable Constants

All constants are defined in **Section 5 (Constants & Configuration)** of `telegram_downloader.ipynb`. Edit them to adjust behavior:

| Constant | Default | Description |
|----------|---------|-------------|
| **Download Settings** | | |
| `CHUNK_SIZE` | 1 MB | Size of each download chunk for large files |
| `LARGE_FILE_THRESHOLD` | 50 MB | Files above this use chunked download with resume |
| `SAVE_EVERY` | 10 | Save tracker to disk every N downloads |
| **Download Delays** | | |
| `INTER_DELAY_TINY` | 0.5s | Delay between downloads for files <1 MB (photos) |
| `INTER_DELAY_SMALL` | 2s | Delay between downloads for files 1-10 MB |
| `INTER_DELAY_MEDIUM` | 3s | Delay between downloads for files 10-50 MB |
| `INTER_DELAY_LARGE` | 5s | Delay between downloads for files 50-500 MB |
| `INTER_DELAY_XLARGE` | 10s | Delay between downloads for files >500 MB |
| **Upload Settings** | | |
| `SEND_DELAY_SMALL` | 3s | Upload cooldown for files <100 MB |
| `SEND_DELAY_MEDIUM` | 10s | Upload cooldown for files 100-500 MB |
| `SEND_DELAY_LARGE` | 20s | Upload cooldown for files >500 MB |
| `UPLOAD_FLOOD_RETRY_CAP` | 10 | Max FloodWait retries per file before skipping |
| `UPLOAD_BATCH_SIZE` | 10 | Max photos per `send_media_group` batch |
| `UPLOAD_CONCURRENT` | 3 | Max parallel file uploads |
| `UPLOAD_SAVE_EVERY` | 10 | Save sent_tracker every N successful uploads |
| **File Naming** | | |
| `FILE_NAME_TEMPLATE` | `""` | Custom naming template (empty = original filename) |

### Custom File Naming

Set `FILE_NAME_TEMPLATE` to customize downloaded filenames:

```python
# Examples:
FILE_NAME_TEMPLATE = ""                              # Default: original filename
FILE_NAME_TEMPLATE = "{date}_{msgid}_{filename}"     # 2026-02-28_123_video.mp4
FILE_NAME_TEMPLATE = "{date_time}_{type}_{msgid}"    # 2026-02-28_143055_video_123.mp4
FILE_NAME_TEMPLATE = "{type}_{date}_{filename}"      # video_2026-02-28_original.mp4
```

Available placeholders:
- `{filename}` — Original filename without extension
- `{ext}` — File extension (`.mp4`, `.jpg`, etc.)
- `{msgid}` — Telegram message ID
- `{date}` — Message date (`YYYY-MM-DD`)
- `{date_time}` — Message datetime (`YYYY-MM-DD_HHMMSS`)
- `{type}` — Media type (`video`, `photo`, `document`, etc.)

---

## Technical Deep Dive

### Upload Speed and FloodPremiumWait

**The problem**: Telegram throttles free accounts during large file uploads via `FLOOD_PREMIUM_WAIT` errors. This is a **server-side rate limit** — no client-side workaround can bypass it.

**How Pyrogram's `save_file()` works**: Files are split into 512KB chunks. 4 worker coroutines upload chunks in parallel via `session.invoke(SaveFilePart(...))`. When a worker hits `FloodPremiumWait`:

| `sleep_threshold` | Worker behavior | Result |
|-------------------|----------------|--------|
| `0` (default) | Worker catches exception, **skips** the chunk, moves to next | **Corrupt file** (missing chunks) |
| `120` (our setting) | Pyrogram sleeps internally, **retries** the chunk | **Correct file** but slow upload |

This is why we set `sleep_threshold=120` — it's critical for upload correctness on free accounts.

**Expected upload times for free accounts** (when throttled):
- Files <100 MB: 1-5 minutes
- Files 100-500 MB: 5-20 minutes
- Files 500 MB-2 GB: 20-60+ minutes
- Progress may pause for 30-120 seconds between chunk batches (Telegram throttle)

### Upload Pipeline Patch (Queue(1) to Queue(16))

Pyrogram's default `asyncio.Queue(1)` only allows 1 chunk in-flight at a time. We monkey-patch to `Queue(16)`, allowing 16 chunks in-flight through 4 workers. This pipelines the upload for **3-8x speed improvement** on non-throttled connections (Premium accounts or small files).

The patch is scoped exclusively to Pyrogram's `save_file` module — no other `asyncio.Queue` usage is affected.

### SQLite Double-Patch Guard

On Colab re-runs (same kernel), the sqlite3 monkey-patch could re-apply on an already-patched function, causing infinite recursion (`RecursionError`). A `_tgdl_patched` flag on the `sqlite3` module prevents this.

### Error Handling Matrix

| Error | Where | Handling |
|-------|-------|----------|
| `FloodWait` | Downloads, uploads, scanning | Sleep `e.value` seconds, retry |
| `FloodPremiumWait` | Uploads (free accounts) | Sleep `e.value` seconds, retry (capped at 10 retries for uploads) |
| `FileReferenceExpired` | Downloads | Re-fetch message via `get_messages()`, retry (up to 3 times) |
| `ChannelPrivate` | Channel access | Skip entire channel |
| `ChatAdminRequired` | Channel access | Skip entire channel |
| `RPCError` | Any API call | Log error, add to failed list, continue next file |
| `KeyboardInterrupt` | Downloads, uploads | Save tracker, add remaining files to retry queue |
| 0-byte file | After download | Auto-detect and remove |
| File > upload limit | Before upload | Skip with warning (2 GB free / 4 GB Premium) |

### Tracker Formats

**download_tracker.json** — Tracks downloaded message IDs per channel:
```json
{
  "-1003730767615": {
    "title": "Channel Name",
    "downloaded_ids": [3, 5, 7, 12, 15],
    "last_updated": "2026-02-28T17:52:55.113520"
  }
}
```

**sent_tracker.json** — Tracks files sent to Saved Messages:
```json
{
  "ChannelName": {
    "files": ["video_3.mp4", "photo_5.jpg"],
    "last_updated": "2026-02-28T18:30:00.000000"
  }
}
```

Both trackers use cached sets (`_ids_set` / `_files_set`) for O(1) duplicate lookups at runtime, stripped before writing to disk.

---

## Known Issues & Warnings

These issues were identified from real-world session logs and are documented here for transparency.

### Issue 1: FLOOD_PREMIUM_WAIT Log Spam During Uploads

**Symptom:** When uploading files on a **free Telegram account**, the console is flooded with hundreds of `ERROR:pyrogram.methods.advanced.save_file` tracebacks like this:

```
ERROR:pyrogram.methods.advanced.save_file:Telegram says: [420 FLOOD_PREMIUM_WAIT_X]
  (caused by "upload.SaveBigFilePart") Pyrogram 2.3.69 thinks: A wait of 13 seconds
  is required
Traceback (most recent call last):
  File ".../pyrogram/methods/advanced/save_file.py", line 109, in worker
    await session.invoke(data)
  ...
pyrogram.errors.exceptions.flood_420.FloodPremiumWait: ...
```

**Impact:** The log becomes unreadable. In a real session uploading 2.25 GB (4 files), this produced ~5,800 lines of traceback output — 95% of the entire log.

**Root cause:** Pyrogram's `save_file()` has 4 internal worker coroutines. Each worker catches `FloodPremiumWait`, logs the full traceback at ERROR level, sleeps, and retries. With `sleep_threshold=120`, the workers correctly retry (no data loss), but each retry produces 11 lines of traceback. For a 1.93 GB file upload on a free account, this means hundreds of retries and thousands of log lines.

**Is it a problem?** **No** — the uploads complete successfully. The tracebacks are Pyrogram's internal logging, not actual errors in the tool. The files are uploaded correctly and completely.

**Status:** Open. Potential fix: suppress the `save_file` logger specifically:
```python
logging.getLogger("pyrogram.methods.advanced.save_file").setLevel(logging.CRITICAL)
```
This would silence the worker tracebacks while keeping other useful Pyrogram warnings.

**Trade-off:** Suppressing these logs means you won't see the per-chunk retry details. For most users this is fine, but for debugging upload issues it could hide useful information.

---

### Issue 2: Free Account Upload Speed Cap

**Symptom:** Upload speeds are capped at approximately **~1.25 MB/s** (~10 Mbit/s) on free Telegram accounts, regardless of your actual internet speed.

**Root cause:** Telegram enforces a **server-side rate limit** via `FLOOD_PREMIUM_WAIT` errors on the `upload.SaveBigFilePart` API call. This is not a bug — it's Telegram's way of incentivizing Premium subscriptions.

**Real-world upload times observed (free account on Google Colab):**

| File | Size | Upload Time | Effective Speed |
|------|------|-------------|-----------------|
| video_129.mp4 | 24 MB | ~4m 30s | ~0.09 MB/s (heavily throttled) |
| video_128.mp4 | 209 MB | ~15m 8s | ~0.23 MB/s |
| video_127.mp4 | 1.93 GB | ~27m 3s | ~1.22 MB/s |
| video_125.mp4 | 91 MB | ~5m | ~0.30 MB/s |

**Note:** The effective speed varies because Telegram throttles in bursts — chunks upload fast, then pause for 10-13 seconds, then resume.

**Status:** Unfixable (server-side). Only Telegram Premium removes this limit.

---

### Issue 3: Pyrogram WARNING Logs During Throttle

**Symptom:** Between the ERROR tracebacks, you also see WARNING messages:

```
WARNING:pyrogram.session.session:[.../my_telegram_session] Waiting for 10 seconds
  before continuing (required by "upload.SaveBigFilePart")
```

**Impact:** Additional log noise, but these are actually **useful** — they confirm Pyrogram is correctly sleeping and retrying (thanks to `sleep_threshold=120`).

**Status:** By design. These are working as intended and confirm correct behavior.

---

### Other Known Limitations

| Limitation | Description |
|-----------|-------------|
| **Single TCP connection per upload** | Pyrogram uses one media session. Multi-connection (like Telegram Desktop) would need a significant rewrite |
| **No parallel file downloads** | Files download sequentially. Parallel would risk more aggressive FloodWait from Telegram |
| **Session file is sensitive** | `my_telegram_session.session` contains your auth token — never share it |
| **Colab session volatility** | Colab sessions can disconnect after ~90 minutes of inactivity. Use the Saved Messages scan to recover tracker state |
| **No date range filter** | Cannot yet filter downloads by date — downloads all files matching the selected types |

---

## Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `pyrofork` | >=2.3.0 | Pyrogram fork — MTProto API client, restricted content support |
| `TgCrypto-pyrofork` | >=1.2.5 | C-based AES encryption for fast MTProto operations |
| `nest_asyncio` | (Colab only) | Patches running event loop in Colab's Jupyter environment |
| `uvloop` | (Colab only) | 2-4x faster async event loop (Linux only) |

---

## Future Work

These were identified during development but not yet implemented:

### High Priority
- **Hash-based deduplication** — Use `file_unique_id` to detect duplicate files across channels
- **Download speed limiter** — Configurable bandwidth cap to reduce FloodWait frequency
- **Proxy/SOCKS5 support** — For users behind firewalls (`Client(proxy=...)`)
- **Export media list to CSV** — Channel inventory without downloading
- **Suppress `save_file` worker tracebacks** — Reduce log noise during uploads (see [Issue 1](#issue-1-flood_premium_wait-log-spam-during-uploads))

### Medium Priority
- **Caption/metadata preservation** — Save message text alongside media files
- **Date range filter** — Download only messages from a specific date range
- **File type filter** — e.g., only `.mp4` videos, skip `.mkv`
- **Parallel file downloads** — Download 2-3 files simultaneously (with FloodWait backoff)
- **Google Drive auto-backup** — Mount Drive on Colab and sync downloads

### Low Priority
- **Progress bar with `tqdm`** — Replace custom progress display
- **Config file for all settings** — Move constants to a `settings.json`
- **Telegram bot notification** — Send download completion alerts via bot
- **Web UI** — Simple local web interface instead of CLI

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `FloodWait` errors during download | Increase `INTER_DELAY_*` constants. Telegram rate-limits API calls |
| `FloodPremiumWait` spam in logs | Normal for free accounts uploading large files. Uploads complete correctly despite the errors — see [Issue 1](#issue-1-flood_premium_wait-log-spam-during-uploads) |
| Upload progress pauses for 30-120s | Normal — Telegram throttles free accounts between chunk batches. The upload WILL resume automatically |
| Very slow uploads (~1 MB/s) | Free accounts are capped at ~1.25 MB/s by Telegram. Only Premium removes this — see [Issue 2](#issue-2-free-account-upload-speed-cap) |
| Very slow downloads (~200 KB/s) | Use Google Colab for ~5-10 MB/s. Free accounts are throttled on local networks |
| `FileReferenceExpired` | Normal for long sessions. The tool auto-retries with fresh references (3 attempts) |
| `ChannelPrivate` error | You were removed from the channel, or it was deleted |
| Session file corrupt | Delete `my_telegram_session.session` and re-authenticate |
| Tracker file corrupt | The tool auto-resets to `{}`. Or delete `download_tracker.json` manually |
| `RecursionError` on Colab | Restart the Colab runtime (Runtime → Restart runtime), then re-run |
| 0-byte files in downloads | The tool auto-detects and removes these. Old 0-byte files are skipped during upload |
| `ModuleNotFoundError: pyrogram` | Run `pip install pyrofork TgCrypto-pyrofork` |
| Colab session disconnects | Re-run the cell. Tracker files persist — the tool skips already-downloaded files. Use Saved Messages scan to recover upload tracker |

---

## Security Notes

- **Never share** `my_telegram_session.session` — it contains your Telegram authentication token. Anyone with this file can access your account.
- **Never share** `tg_config.json` — it contains your API credentials (`api_id` and `api_hash`).
- This tool uses your **personal Telegram account**, not a bot. All actions appear as if you performed them.
- All communication goes through Telegram's **official MTProto protocol** — the same protocol every Telegram client uses.
- Protected content access works because you are an **authorized member** of the channel. The tool does not bypass any authentication or encryption.
- Downloaded files are stored **locally** (or in Colab session storage). No data is sent to any third party.

---

## License & Disclaimer

This project is provided **as-is** for **educational and research purposes only**.

The authors and contributors:
- Make **no warranties** about the software's fitness for any purpose
- Accept **no liability** for any damages arising from use of this software
- Are **not responsible** for how users choose to use this tool
- Do **not endorse** downloading copyrighted content without authorization

Users are solely responsible for:
- Complying with Telegram's Terms of Service
- Respecting copyright and intellectual property laws
- Ensuring their use is legal in their jurisdiction

**By using this tool, you acknowledge that you understand and accept these terms.**
