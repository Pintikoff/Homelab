# Jellyfin — Self-Hosted Media Server

## What it is

Jellyfin is a free, open-source media server. It indexes a media library (movies, shows, music) and streams it to clients over the network, with a web UI, apps for most platforms, and support for transcoding video on the fly for devices that can't play a file directly. It's a fully self-hosted alternative to Plex or Emby, with no account or cloud dependency.

## Why use it

- Single place to organize and stream a personal media collection to any device on (or off) the local network
- No subscription, no vendor lock-in, data stays on hardware you control
- Fetches posters, descriptions, and metadata automatically once files are named correctly
- Single container, no database or reverse proxy required to get started, making it a good first Docker service to learn on

## Docker Compose setup

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin
    container_name: jellyfin
    ports:
      - "8096:8096"
    volumes:
      - ./media:/media
    restart: unless-stopped
```

```bash
docker compose up -d
```

- `ports: "8096:8096"` maps host port 8096 to the container's internal port 8096. Jellyfin listens on 8096 inside its own isolated container network by default and isn't reachable from outside the container without this mapping.
- `volumes: ./media:/media` mounts a local folder into the container at `/media`. Files placed in `./media` on the host become visible to Jellyfin inside the container at `/media`. When adding a library in the Jellyfin UI, point it at `/media` (the path as seen *inside* the container), not the host path.

Open `http://<server-ip>:8096` to run first-time setup: language, admin account, then **Add Media Library**.

## File and folder naming

Jellyfin's metadata matching works much better with clean, structured names. Recommended layout, one folder per title:

```
Movie Title (Year)/
├── Movie Title (Year).mkv
└── Movie Title (Year).en.srt
```

- The year in parentheses is the main anchor Jellyfin uses to find the correct match.
- Spaces read more reliably than dots in filenames.
- External subtitles are picked up automatically when their filename matches the video's filename, with a language code appended (`.en.srt`, `.ru.srt`, etc.) — both files need the same base name.

## Refreshing the library

New files dropped into the mounted media folder don't need a container restart. Trigger a rescan instead:

**Dashboard → Libraries → Scan All Libraries**

A restart (`docker compose restart` or `docker compose up -d` after editing the compose file) is only needed after changing the compose configuration itself (ports, volumes, environment variables) — not after adding new media files.

## Notes on hardware

Direct playback (the client plays the file as-is) is effectively free on CPU. Transcoding (Jellyfin re-encodes the video on the fly, because the client can't play the source format/resolution/codec) is CPU-intensive. On older or low-power hardware, prefer H.264/1080p sources over HEVC/4K where possible, since HEVC decoding usually has no hardware acceleration on older CPUs and forces a costly software transcode if a client can't play it directly.
