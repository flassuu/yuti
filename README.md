# yuti

A minimal, yt-dlp-powered CLI for managing offline YouTube Music playlists — inspired by `spotdl sync`.

Register playlists once with all your preferred flags, run `yuti sync` whenever you want updates, and only new tracks get downloaded. Everything is stored as plain text files in `~/.config/yuti/`, compatible with Syncthing, rsync, or any backup tool.

---

## Features

- Register playlists with a name, URL, optional custom path, and saved per-playlist flags
- Smart sync via yt-dlp's download archive — already-downloaded tracks are always skipped
- Compact per-track status display during sync; full raw output available with `--nerd`
- M3U playlist generation after sync
- Lyrics download (LRC format)
- Automatic MPD library update via `mpc update` (optional)
- Configurable audio bitrate (128 / 192 / 256 / 320 kbps)
- One-shot `download` command with no config needed
- Persistent sync history log with timestamps and per-run stats
- XDG-compliant config layout — no files outside `~/.config/yuti/`

---

## Requirements

### Required

| Tool     | Purpose                           | Arch install         |
|----------|-----------------------------------|----------------------|
| `yt-dlp` | Downloading and audio extraction  | `pacman -S yt-dlp`   |
| `jq`     | Reading and writing playlist JSON | `pacman -S jq`       |
| `ffmpeg` | Audio conversion to MP3           | `pacman -S ffmpeg`   |

### Optional

| Tool           | Purpose                                      | Arch install                |
|----------------|----------------------------------------------|-----------------------------|
| `atomicparsley`| Embedding album art into MP3 files           | `pacman -S atomicparsley`   |
| `mpc`          | Triggering MPD library refresh after sync    | `pacman -S mpc`             |

Without `atomicparsley`, yt-dlp will skip thumbnail embedding silently — everything else works normally. Without `mpc`, the `--mpcu` flag has no effect.

---

## Installation

```bash
git clone https://github.com/flassuu/yuti.git
cd yuti
chmod +x yuti
sudo cp yuti /usr/local/bin/
```

Or without cloning:

```bash
curl -o ~/.local/bin/yuti https://raw.githubusercontent.com/flassuu/yuti/main/yuti
chmod +x ~/.local/bin/yuti
```

---

## Usage

```
yuti <command> [arguments] [flags]
```

### Commands

| Command                           | Description                                              |
|-----------------------------------|----------------------------------------------------------|
| `add <name> <url> [path] [flags]` | Register a new playlist; flags are saved to its config   |
| `update <name> [url] [path] [flags]` | Update URL, path, or saved flags for an existing playlist |
| `remove <name>`                   | Remove playlist config and archive (not music files)     |
| `sync <name> [flags]`             | Download new tracks; flags override saved config per run |
| `download <url\|name> [path] [flags]` | One-shot download without creating a sync config     |
| `sync-list`                       | List all playlists with paths, flags, and last sync time |
| `sync-history [N]`                | Show the last N sync log entries (default: 10)           |

### Flags

| Flag                   | Description                                          | Default |
|------------------------|------------------------------------------------------|---------|
| `-q, --quality <kbps>` | Audio bitrate: `128` / `192` / `256` / `320`         | `192`   |
| `--m3u`                | Generate an `.m3u` playlist file after sync          | off     |
| `--lyrics`             | Download lyrics (LRC) alongside tracks               | off     |
| `--mpcu`               | Run `mpc update` after sync if mpc is installed      | off     |
| `--nerd`               | Show full raw yt-dlp output instead of compact view  | off     |
| `-h, --help`           | Show help and exit                                   |         |
| `-v, --version`        | Show version and exit                                |         |

### Saving flags per playlist

Flags passed to `add` (or `update`) are stored in the playlist's JSON config and reused automatically every time you run `sync`. Runtime flags passed directly to `sync` override the saved values for that run only.

```bash
# Register a playlist with 320 kbps, M3U generation, and lyrics always on
yuti add lofi "https://music.youtube.com/playlist?list=PLxxx" --quality 320 --m3u --lyrics

# From now on, every sync uses 320 kbps + M3U + lyrics automatically
yuti sync lofi

# Override quality just for this run, everything else from saved config
yuti sync lofi --quality 128
```

---

## Quick Start

```bash
# Register a playlist (saves to ~/Music/lofi/ by default)
yuti add lofi "https://music.youtube.com/playlist?list=PLxxx"

# Register with custom path and saved flags
yuti add chill "https://music.youtube.com/playlist?list=PLyyy" ~/Music/chill --quality 320 --m3u

# Download new tracks
yuti sync lofi

# Download with full yt-dlp output visible
yuti sync lofi --nerd

# One-shot download to current directory (no config created)
yuti download "https://music.youtube.com/playlist?list=PLzzz"

# One-shot download to a specific path with flags
yuti download "https://music.youtube.com/playlist?list=PLzzz" ~/downloads/party --quality 256 --m3u

# See all playlists and when they were last synced
yuti sync-list

# View the 20 most recent sync events
yuti sync-history 20
```

---

## How it Works

### Playlist configs

Each registered playlist is stored as a JSON file in `~/.config/yuti/playlists/`:

```json
{
  "name": "lofi",
  "url": "https://music.youtube.com/playlist?list=PLxxx",
  "path": "/home/user/Music/lofi",
  "created_at": "2026-05-04T14:00:00",
  "updated_at": "2026-05-04T14:00:00",
  "flags": {
    "quality": "320",
    "m3u": true,
    "lyrics": false,
    "mpcu": false
  }
}
```

### Download archives

yt-dlp's `--download-archive` maintains a list of video IDs already downloaded in `~/.config/yuti/archives/<name>.txt`. On every sync, yt-dlp checks this file and skips matching tracks — regardless of whether the audio file still exists on disk.

> Note: if you delete a track and want to re-download it, remove its line from the archive file manually, or delete the whole archive file to reset the playlist.

### Sync output modes

**Default (compact):** A single updating line per track shows what yt-dlp is currently doing. Each track prints one final colored status line:

```
  [Done]         Nujabes - Feather
  [Skipped]      Four Tet - Sing
  [Error]        Unknown Track
```

**Nerd mode (`--nerd`):** Full raw yt-dlp output is shown, including download progress bars, conversion steps, and all diagnostic messages.

### Sync history

Every `sync` run appends a line to `~/.config/yuti/sync_history.log`:

```
2026-05-04T14:32:11 | lofi | downloaded=3 skipped=12 errors=0 | success
```

The `sync-history` command reads this file and displays recent entries newest-first.

### M3U generation

When `--m3u` is active, yuti scans the output directory after sync and writes a `<name>.m3u` file listing all audio files found. This is compatible with MPD, VLC, and most other players.

---

## Config Layout

```
~/.config/yuti/
├── playlists/
│   ├── lofi.json
│   └── chill.json
├── archives/
│   ├── lofi.txt          # yt-dlp download archive (track deduplication)
│   └── chill.txt
└── sync_history.log      # append-only sync event log
```

---

## yt-dlp Options Used

| Option                  | Purpose                                             |
|-------------------------|-----------------------------------------------------|
| `--extract-audio`       | Strip video stream, keep audio only                 |
| `--audio-format mp3`    | Convert to MP3                                      |
| `--audio-quality <N>K`  | Set bitrate from `--quality` flag                   |
| `--embed-thumbnail`     | Embed album art (requires `atomicparsley` for MP3)  |
| `--add-metadata`        | Embed title, artist, and other tags                 |
| `--download-archive`    | Skip already-downloaded tracks                      |
| `--ignore-errors`       | Continue syncing if a single track fails            |
| `--write-subs`          | Download subtitle/lyrics files (`--lyrics` flag)    |
| `--convert-subs lrc`    | Convert lyrics to LRC format (`--lyrics` flag)      |

---

## Offline Sync with Syncthing

All music and config files are plain files on disk, so any file sync tool works:

1. Set your playlist path to a folder managed by Syncthing (e.g. `~/Sync/Music/lofi`)
2. Run `yuti sync lofi` on your main machine
3. New tracks appear on all other synced devices automatically

The `archives/` directory should also be synced if you want to avoid re-downloading on secondary devices.

---

## License

MIT
>>>>>>> f62e602 (Initial commit)
