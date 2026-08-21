# Saidawan Radio

[radio.2mohsen.ir](https://radio.2mohsen.ir/)

**Saidawan Radio** is a distributed, GitHub-based music streaming system that collects audio from a Telegram channel, extracts and maintains metadata, stores the audio files across multiple GitHub repositories, and serves them through a static web radio application.

The system is designed around a simple idea:

> **Telegram is the source of the music, GitHub is the storage and delivery layer, and the static website is the player.**

There is no traditional backend server or database required for the public player. GitHub repositories act as both the storage layer and part of the data-delivery infrastructure.

## Architecture

```text
                       ┌─────────────────────┐
                       │   Telegram Channel  │
                       │      @saidawan      │
                       └──────────┬──────────┘
                                  │
                    Read messages / audio metadata
                                  │
                                  ▼
                       ┌─────────────────────┐
                       │     audio-data      │
                       │                     │
                       │ Telegram metadata   │
                       │ → JSON database     │
                       └──────────┬──────────┘
                                  │
                           data.json
                                  │
              ┌───────────────────┼───────────────────┐
              │                   │                   │
              ▼                   ▼                   ▼
        ┌───────────┐       ┌───────────┐       ┌───────────┐
        │  pocket01 │       │  pocket02 │  ...  │  pocket06 │
        │           │       │           │       │           │
        │ Music     │       │ Music     │       │ Music     │
        │ Covers    │       │ Covers    │       │ Covers    │
        │ Metadata  │       │ Metadata  │       │ Metadata  │
        └─────┬─────┘       └─────┬─────┘       └─────┬─────┘
              │                   │                   │
              │      Raw GitHub audio URLs            │
              └───────────────────┼───────────────────┘
                                  │
                                  ▼
                    ┌────────────────────────┐
                    │saidawan-radio.github.io│
                    │                        │
                    │ Static Web Radio       │
                    │ Music Player           │
                    └────────────────────────┘
```

## How It Works

The system is divided into four main stages:

1. **Collect metadata from Telegram**
2. **Download and store audio files**
3. **Generate publicly accessible GitHub URLs**
4. **Load the music from the static web application**

---

## 1. Telegram as the Source

The original music is published in a Telegram channel.

The `audio-data` repository uses **Telethon** to connect to Telegram and inspect messages in the configured channel.

For every audio message, the system extracts information such as:

* Title
* Performer / artist
* Duration
* Telegram message ID
* Channel information
* Message text
* MIME type
* File size
* Telegram document ID
* Publication date
* Last edit date
* Generated filename

The `AudioDetail` class is responsible for converting Telegram audio messages into structured metadata.

The generated filename follows a predictable structure based on the internal ID, channel, title, performer, and date.

This gives every audio file a stable identity that can be used by the rest of the system.

---

# 2. `audio-data`

Repository:

[`saidawan-radio/audio-data`](https://github.com/saidawan-radio/audio-data)

`audio-data` is the **central metadata repository**.

It does not primarily store the music files. Instead, it maintains the structured information describing the music available from Telegram.

Its main components include:

* `AudioDetail.py`
* `FetchDetail.py`
* `fetch_and_write_detail.py`
* `config.py`
* `data.json`
* GitHub Actions workflow

The metadata is stored in JSON using structures such as:

```text
audio_info
map_msg_id
general_info
```

`map_msg_id` maps Telegram message IDs to the internal IDs used by the system, while `audio_info` contains the actual audio metadata.

### Fetching metadata

`fetch_and_write_detail.py` creates a Telegram client using Telethon and fetches audio messages from the configured channel.

The metadata-fetching process:

```text
Telegram
   │
   ├── Find audio messages
   │
   ├── Extract AudioDetail
   │
   ├── Assign internal ID
   │
   ├── Detect updated messages
   │
   └── Write data.json
```

The system also detects edits to existing Telegram messages and updates their metadata while preserving important identifiers.

### Automated updates

The repository contains a GitHub Actions workflow named `fetch-data`.

It can be manually triggered and is also scheduled to run every two days.

The workflow:

1. Fetches audio metadata.
2. Writes the updated JSON.
3. Commits and pushes the changes back to the repository.

This makes `audio-data/data.json` the continuously updated **source of truth for the music catalog**.

---

# 3. `audio-source`

Repository:

[`saidawan-radio/audio-source`](https://github.com/saidawan-radio/audio-source)

`audio-source` is used to collect and combine audio metadata from external source files.

It contains:

* `colector.py`
* `fetch_source.sh`
* `source.json`
* `sources-url`
* `file-sources/`

The `fetch_source.sh` script reads URLs from `sources-url`, downloads each JSON source into `file-sources/`, and then runs the collector.

The collector merges the audio information from those source files into `source.json` and calculates aggregate information such as:

* Total duration
* Highest/latest internal audio ID

This repository therefore acts as a **metadata/source aggregation layer** separate from the main Telegram metadata collector.

---

# 4. The `pocketXX` Repositories

The organization contains six storage repositories and can be add more pocket repo if needed:

* [`pocket01`](https://github.com/saidawan-radio/pocket01)
* [`pocket02`](https://github.com/saidawan-radio/pocket02)
* [`pocket03`](https://github.com/saidawan-radio/pocket03)

These repositories form the **distributed music storage layer**.

Each pocket repository contains directories such as:

```text
musics/
covers/
```

as well as the Python code and GitHub Actions used to download and process audio. The repositories follow the same general structure.

## Why multiple pockets?

Instead of putting the entire music collection into one repository, the collection is divided across multiple repositories.

Conceptually:

```text
                    Music Collection
                           │
          ┌────────────────┼────────────────┐
          │                │                │
       pocket01         pocket02         pocket03
          │                │                │
       pocket04         pocket05         pocket06
```

This distributes the storage and Git history across several repositories.

The individual pocket repositories are therefore primarily **storage buckets for the actual audio files**.

---

# 5. Downloading Music into a Pocket

A pocket repository receives the canonical metadata from `audio-data`.

For example, the `pocket01` workflow downloads `data.json` from the configured `audio-data` URL before starting the download process.

The workflow then runs the audio downloader with the Telegram credentials and configuration.

The downloader determines which internal audio ID should be downloaded next:

```text
audio-data/data.json
          │
          ▼
last downloaded ID
          │
          ▼
next audio item
          │
          ▼
Telegram message
          │
          ▼
download audio
          │
          ▼
pocketXX/musics/
```

The downloader retrieves the corresponding Telegram message and downloads its audio media.

---

# 6. GitHub Raw URLs

After downloading an audio file, the pocket system generates its GitHub Raw URL.

The downloader adds this URL to the audio metadata as `github_url`.

This is an important part of the architecture.

The website does not need a custom audio server.

Instead:

```text
Web Browser
     │
     │ HTTP
     ▼
GitHub Raw
     │
     ▼
.opus audio file
```

GitHub becomes the static file delivery layer.

---

# 7. Audio Conversion

The pocket repositories also contain an audio conversion stage.

The purpose of this stage is to process the downloaded audio into the specific and optimize format, particularly Opus format.

The final audio files are stored in the pocket's `musics/` directory.

---

# 8. Cover Images

The pocket repositories also maintain album/track covers.

The `extract-cover.yml` workflow watches for Opus files and extracts their embedded cover images.

The cover filename is associated with the numeric audio ID extracted from the audio filename.

This allows the web application to associate an image with each track.

---

# 9. Keeping Metadata Up to Date

The pocket repositories contain another workflow called `data-update`.

This workflow periodically retrieves the latest metadata from `audio-data` and updates metadata stored by the pocket.

It is scheduled to run every two days and can also be manually triggered.

This is important because Telegram messages can be edited.

For example:

```text
Telegram
   │
   │ title changed
   ▼
audio-data
   │
   │ updated metadata
   ▼
pocketXX
```

The `dataupdate.py` script compares the edit dates of stored metadata with the metadata from `audio-data` and updates fields such as:

* Title
* Performer
* Message
* Edit date

when the source metadata is newer.

---

# 10. `saidawan-radio.github.io`

Repository:

[`saidawan-radio/saidawan-radio.github.io`](https://github.com/saidawan-radio/saidawan-radio.github.io)

This is the **public web application**.

It is deployed as a static GitHub Pages website:

[`saidawan-radio.github.io`](https://saidawan-radio.github.io)

The organization therefore does not need a conventional backend server to host the public player.

---

# 11. The Web Player

The website acts as the presentation layer of the entire system.

Its responsibilities are essentially:

```text
Load metadata
     ↓
Display music
     ↓
Display covers
     ↓
Create playlist / radio experience
     ↓
Play audio from GitHub
```

The frontend is a React-based application and uses Material UI components in its compiled bundle.

The actual audio files remain in the `pocketXX` repositories rather than inside the website repository.

---

# Complete Data Flow

The entire organization can be understood as one pipeline:

```text
┌───────────────────────────┐
│       Telegram Channel    │
│                           │
│  New audio message        │
└─────────────┬─────────────┘
              │
              │ Telethon
              ▼
┌───────────────────────────┐
│        audio-data         │
│                           │
│  AudioDetail              │
│  FetchDetail              │
│  data.json                │
└─────────────┬─────────────┘
              │
              │ JSON metadata
              ▼
┌────────────────────────────────────────────┐
│              pocketXX                     │
│                                            │
│  Download Telegram audio                   │
│          ↓                                 │
│  Convert / tag audio                       │
│          ↓                                 │
│  Store in musics/                          │
│          ↓                                 │
│  Extract cover                             │
│          ↓                                 │
│  Store in covers/                          │
│          ↓                                 │
│  Generate GitHub Raw URL                   │
└───────────────────┬────────────────────────┘
                    │
                    │ HTTPS
                    ▼
┌────────────────────────────────────────────┐
│        saidawan-radio.github.io             │
│                                            │
│            Static Web Radio                │
│                                            │
│  Metadata → Playlist → Audio Player        │
└────────────────────────────────────────────┘
```

---

# Repository Responsibilities

| Repository                 | Responsibility                                                |
| -------------------------- | ------------------------------------------------------------- |
| `audio-data`               | Collects and maintains canonical audio metadata from Telegram |
| `audio-source`             | Aggregates metadata from external JSON sources                |
| `pocket01`                 | Music storage + download/processing pipeline                  |
| `pocket02`                 | Music storage + download/processing pipeline                  |
| `pocket03`                 | Music storage + download/processing pipeline                  |
| `pocket04`                 | Music storage + download/processing pipeline                  |
| `pocket05`                 | Music storage + download/processing pipeline                  |
| `pocket06`                 | Music storage + download/processing pipeline                  |
| `saidawan-radio.github.io` | Static web radio and music player                             |

The organization currently exposes all nine repositories publicly.

---

# Automation

GitHub Actions are an important part of the architecture.

The system uses Actions to automate the lifecycle of the music collection.

### Metadata pipeline

```text
Scheduled GitHub Action
        │
        ▼
Connect to Telegram
        │
        ▼
Fetch audio metadata
        │
        ▼
Update data.json
        │
        ▼
Commit + Push
```
