# Immich Job Daemon

**[Русский](README.md) | English**

[![Docker Build](https://github.com/alternativniy/immich-job-daemon/actions/workflows/docker-build.yml/badge.svg)](https://github.com/alternativniy/immich-job-daemon/actions/workflows/docker-build.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Docker Pulls](https://img.shields.io/docker/pulls/alternativniy/immich-job-daemon)](https://github.com/alternativniy/immich-job-daemon/pkgs/container/immich-job-daemon)

A daemon for managing Immich job queue. Automatically manages jobs by priority, allowing you to run a specified number of jobs simultaneously.

---

## Features


- 🐧 Based on Alpine Linux (minimal image size)
- 🔄 Automatic job priority management
- ⚙️ Configurable number of concurrent jobs
- 🕒 Configurable active hours for job processing
- 📣 NTFY notifications support
- 🔒 Runs as non-privileged user
- 🌐 Configuration via environment variables

---

## Job Priority

Jobs are processed in the following priority order:

1. sidecar
2. metadataExtraction
3. storageTemplateMigration
4. thumbnailGeneration
5. smartSearch
6. duplicateDetection
7. faceDetection
8. facialRecognition
9. videoConversion
10. other jobs

---

## Usage

### Using Pre-built Image from GitHub Container Registry

```bash
docker run -d \
  --name immich-job-daemon \
  -e IMMICH_URL=http://your-immich-server:2283 \
  -e API_KEY=your_api_key_here \
  -e MAX_CONCURRENT_JOBS=2 \
  -e POLL_INTERVAL=10 \
  -e START_HOUR=0 \
  -e END_HOUR=23 \
  -e NTFY_TOPIC=your_ntfy_topic \
  -e NTFY_BASE_URL=https://ntfy.sh \
  --restart unless-stopped \
  ghcr.io/alternativniy/immich-job-daemon:latest
```

### Build Image Locally

#### Docker Run

```bash
docker build -t immich-job-daemon .

docker run -d \
  --name immich-job-daemon \
  -e IMMICH_URL=http://your-immich-server:2283 \
  -e API_KEY=your_api_key_here \
  -e MAX_CONCURRENT_JOBS=2 \
  -e POLL_INTERVAL=10 \
  -e START_HOUR=0 \
  -e END_HOUR=23 \
  -e NTFY_TOPIC=your_ntfy_topic \
  -e NTFY_BASE_URL=https://ntfy.sh \
  --restart unless-stopped \
  immich-job-daemon
```

### Docker Compose

1. Edit `docker-compose.yml`:
  ```yaml
   services:
     immich-job-daemon:
       image: ghcr.io/alternativniy/immich-job-daemon:latest
       # Or build locally:
       # build: .
       container_name: immich-job-daemon
       restart: unless-stopped
       environment:
         - IMMICH_URL=http://127.0.0.1:2283
         - API_KEY=your_api_key_here
         - MAX_CONCURRENT_JOBS=2
         - POLL_INTERVAL=10
         - START_HOUR=0
         - END_HOUR=23
         - NTFY_TOPIC=your_ntfy_topic
         - NTFY_BASE_URL=https://ntfy.sh
       depends_on:
         - immich-server
       networks:
         - immich_network
  ```
2. Start the container:
  ```bash
   docker-compose up -d
  ```

> **📝 Note:**The daemon depends on `immich-server` and must be in the same Docker network to access the API.

---

## Environment Variables


| Variable              | Description                                                 | Default                 | Required |
| --------------------- | ----------------------------------------------------------- | ----------------------- | -------- |
| `IMMICH_URL`          | Immich server URL                                           | `http://127.0.0.1:2283` | No       |
| `API_KEY`             | Immich API key with `job.read` and `job.create` permissions | -                       | **Yes**  |
| `MAX_CONCURRENT_JOBS` | Number of jobs running concurrently                         | `1`                     | No       |
| `POLL_INTERVAL`       | Polling interval in seconds                                 | `10`                    | No       |
| `START_HOUR`          | Start hour for job processing (24-hour format)              | `0`                     | No       |
| `END_HOUR`            | End hour for job processing (24-hour format)                | `23`                    | No       |
| `NTFY_TOPIC`          | NTFY topic for notifications                                | -                       | No       |
| `NTFY_BASE_URL`       | NTFY server URL                                             | `https://ntfy.sh`       | No       |
| `NTFY_ACCESS_TOKEN`   | NTFY access token for authentication                        | -                       | No       |
| `NTFY_BASIC_AUTH`     | NTFY basic auth for authentication                          | -                       | No       |
| `NTFY_TITLE`          | Title for NTFY notifications                                | -                       | No       |
| `NTFY_PRIORITY`       | Priority level for NTFY notifications                       | -                       | No       |
| `NTFY_TAGS`           | Tags for NTFY notifications                                 | -                       | No       |
| `NTFY_CLICK_ACTION`   | Click action URL for NTFY notifications                     | -                       | No       |
| `NTFY_ICON_URL`       | Icon URL for NTFY notifications                             | -                       | No       |


---

## Configuration File for NTFY

You can also use a configuration file (`.ntfy_config`) to set NTFY parameters. Example:

```ini
TOPIC=your_ntfy_topic
BASE_URL=https://ntfy.sh
ACCESS_TOKEN=your_access_token
TITLE=Immich Job Daemon
PRIORITY=3
TAGS=camera
```

---

## Getting API Key

1. Log in to Immich web interface
2. Go to **Account Settings** → **API Keys**
3. Create a new API key with required permissions:
   - ✅ `job.read` - to read job status
   - ✅ `job.create` - to manage jobs (pause/resume)
4. Copy and use it in the `API_KEY` variable

> **⚠️ Important:** API key must have `job.read` and `job.create` permissions, otherwise the daemon won't be able to manage jobs.

---

## Logs

View container logs:

```bash
docker logs -f immich-job-daemon
```

---

## How It Works

The daemon runs every N seconds (configurable via `POLL_INTERVAL`):

1. Checks if the current time is within the active hours (configurable via `START_HOUR` and `END_HOUR`)
2. If outside active hours, it skips processing and waits for the next interval
3. If within active hours:
  - Fetches all jobs from Immich API
  - Checks for actively running jobs (active &gt; 0)
  - If there are active jobs - continues their execution until completion (does not interrupt)
  - If all jobs are paused - finds the first N jobs from the priority list (where N = `MAX_CONCURRENT_JOBS`) that have tasks in queue
  - Resumes selected jobs
  - Pauses all other managed jobs

This allows efficient server resource management by processing jobs sequentially or in parallel according to priority, **without interrupting already running jobs**.

---

### Active Hours

The daemon can be configured to only process jobs during specific hours of the day using `START_HOUR` and `END_HOUR` (24-hour format).

**Examples:**

- `START_HOUR=0` and `END_HOUR=23` - runs all day (default)
- `START_HOUR=8` and `END_HOUR=18` - runs only between 08:00 and 18:00
- `START_HOUR=22` and `END_HOUR=6` - runs overnight from 22:00 to 06:00

---

### 🔄 Priority Logic

**Important:** The daemon does not interrupt running jobs! This is critical for jobs that generate data for other jobs.

**Example:**

- Job `thumbnailGeneration` is running (priority 4)
- Data appears for `metadataExtraction` (priority 2, higher)
- The daemon **WILL NOT interrupt** `thumbnailGeneration`, lets it finish
- After all active jobs complete, it will start `metadataExtraction` by priority

---

### NTFY Notifications

The daemon supports sending notifications via NTFY for important events. Configure using environment variables or a `.ntfy_config` file.

**Example Notification:**

- A notification is sent when the daemon starts processing jobs based on active hours.

---

**Usage Examples:**

- `MAX_CONCURRENT_JOBS=1` - jobs run strictly sequentially (default)
- `MAX_CONCURRENT_JOBS=2` - two jobs can run simultaneously
- `MAX_CONCURRENT_JOBS=3` - three jobs can run simultaneously
- `START_HOUR=22` and `END_HOUR=6` - only process jobs overnight
- `NTFY_TOPIC=my_immich_topic` - send notifications to a specific NTFY topic

---

## Requirements

- Docker or Docker Compose
- **Running Immich server** (`immich-server` container)
- Access to Immich API
- Valid Immich API key with `job.read` and `job.create` permissions
- Container must be in the same Docker network as Immich

---

## License

MIT