# Discord Object Store

Use a Discord channel as a lightweight, resumable object store with chunked,
compressed, and optionally encrypted archives. This repo includes:
- `bot_server.py`: Discord bot that uploads/downloads chunks, tracks history,
  and can resume failed transfers.
- `slice_and_assemble.py`: slicer/assembler that creates encrypted chunks and
  restores files from a manifest.
- `archive_card.py`: archive card metadata + embed helpers.

## Features
- Chunking to fit Discord attachment limits (~9.5 MB default).
- AES-256-GCM encryption per chunk with PBKDF2 key derivation.
- Gzip compression (level 9) before encryption.
- Resumable uploads (`!resume`) and idempotent downloads (`!download`).
- Archive cards with metadata, progress, and a dedicated thread per archive.
- Log file backup to a Discord log channel (optional).
- Archive search, verification, and admin cleanup tools.

## How It Works
1) `slice_and_assemble.py` slices a folder into encrypted `.bin` chunks plus a
   JSON manifest. Chunks land in `DISCORD_DRIVE_UPLOAD_PATH`.
2) `!upload` creates an archive card in the archive channel and a thread to hold
   the chunks, uploads each file concurrently, and deletes local chunks after
   successful upload.
3) A local log file tracks archive ID, lot number, message IDs, and status. If a
   log channel is configured, it backs up the log file on every update.
4) `!download` fetches attachments from the archive thread (or legacy message
   IDs), saves them into `[Archive] #DDMMYY-XX` folders, and resumes by skipping
   files already on disk.
5) `slice_and_assemble.py` reassembles the archive using the manifest and
   decrypts/decompresses each chunk. On a clean restore, the chunks folder is
   deleted.

## Security Model
- **Encryption**: AES-256-GCM per chunk.
- **Key derivation**: PBKDF2-HMAC-SHA256 with 600,000 iterations.
- **Salt/nonce**: per-chunk random 16-byte salt + 12-byte nonce.
- **Compression**: gzip level 9 before encryption.
- **Important**: if `USER_KEY` is not set, the slicer refuses to create chunks,
  and the bot will upload unencrypted files if you manually place them in the
  upload folder.

## Requirements
- Python 3.10+
- `discord.py` and `cryptography`
- A Discord bot token with Message Content intent enabled
- A Discord server with channels the bot can read/write attachments
- Default paths assume macOS + external drive mounted at `/Volumes/Local Drive`

## Setup
1) Install dependencies:
```bash
python -m venv .venv
source .venv/bin/activate
pip install -U discord.py
pip install -r requirements.txt
```

2) Create `.env` in the repo root:
```bash
DISCORD_BOT_TOKEN=your-bot-token
DISCORD_DRIVE_ARCHIVE_CHANNEL_ID=123456789012345678
USER_KEY=your-strong-passphrase
```

3) Optional configuration:
```bash
# Optional channels
DISCORD_DRIVE_LOG_CHANNEL_ID=123456789012345678
DISCORD_DRIVE_STORAGE_CHANNEL_ID=123456789012345678
DISCORD_DRIVE_DATABASE_CHANNEL_ID=123456789012345678

# Optional paths (default is /Volumes/Local Drive/DiscordDrive/...)
DISCORD_DRIVE_EXTERNAL_DRIVE_PATH=/Volumes/Local Drive
DISCORD_DRIVE_UPLOAD_PATH=/Volumes/Local Drive/DiscordDrive/Uploads
DISCORD_DRIVE_DOWNLOAD_PATH=/Volumes/Local Drive/DiscordDrive/Downloads
DISCORD_DRIVE_LOG_FILE=/Users/you/discord-drive-history.json
```

## Usage
### 1) Slice files into chunks (recommended for large files)
```bash
python slice_and_assemble.py
```
Choose option 1 and point to a folder of files. Chunks + manifest are written
to the upload folder.

### 2) Start the bot
```bash
python bot_server.py
```
The bot loads logs, restores from the archive channel if available, and waits
for commands.

### 3) Discord commands
- `!upload` — Upload all `.bin` and manifest files in the upload folder.
- `!download #DDMMYY-01 [#DDMMYY-02]` — Download one archive or a range.
- `!resume [archive_id]` — Retry missing chunks for the last failed archive or a specific ID.
- `!status` — Show drive, queue size, encryption status, and last archive info.
- `!history` — Show the 5 most recent archives.
- `!archives [query]` — List or search recent archives by ID/filename.
- `!verify <archive_id>` — Verify that all chunks exist in the archive thread.
- `!rebuild-log` — Rebuild the local log from archive cards (admin).
- `!migrate-legacy` — Migrate legacy log entries to archive cards (admin).
- `!cleanup <archive_id>` — Delete the archive thread + mark archive deleted (admin).
- `!help` — In-channel help.

### 4) Reassemble downloads
```bash
python slice_and_assemble.py
```
Choose option 2 and select the downloaded archive folder, e.g.
`[Archive] #120225-01` inside the download folder. Files are restored next to
that folder; the chunks folder is deleted if restoration is clean.

## Why It Works
- Discord reliably stores attachments and threads, giving you a low-cost,
  always-available blob store.
- The manifest preserves ordering, chunk list, and original file sizes, so the
  assembler can reconstruct files deterministically.
- AES-GCM ensures confidentiality and integrity per chunk; PBKDF2 protects
  the user key from trivial brute-force.
- The local log + archive cards provide a durable index that enables resume,
  verification, and recovery even after partial failures.

## Tips & Troubleshooting
- Keep the log file intact; it maps lot numbers to archive IDs.
- If the drive path is missing and you use defaults, the bot warns but will still
  create folders locally—verify paths before uploading.
- Uploads delete local chunk files after success; keep a backup if needed.
- Downloads are resumable: re-run `!download` to fill missing pieces.
- If decryption fails, confirm `USER_KEY` matches the one used to slice.
