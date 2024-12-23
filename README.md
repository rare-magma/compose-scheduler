# Docker-Compose scheduler

Simple and lightweight service which can execute `docker compose run ...` services from the same file based on cron
expression.

Features:

- Zero-configuration by-default
- Designed for docker compose (auto-detect, respects namespace)
- HTTP notifications with retries

Inspired by [ofelia](https://github.com/mcuadros/ofelia).

```yaml
services:
  web:
    image: "nginx"
    labels:
      - "net.reddec.scheduler.cron=@daily"
      - "net.reddec.scheduler.exec=nginx -s reload"

  date:
    image: busybox
    restart: "no"
    labels:
      - "net.reddec.scheduler.cron=* * * * *"

  scheduler:
    image: ghcr.io/reddec/compose-scheduler:1.0.0
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
```

Supports two modes:

- plain `docker compose run`
- exec command inside service (extra label `net.reddec.scheduler.exec`)

## Usage

```text
Application Options:
      --project=              Docker compose project, will be automatically detected if not set [$PROJECT]

HTTP notification:
      --notify.url=           URL to invoke [$NOTIFY_URL]
      --notify.retries=       Number of additional retries (default: 5) [$NOTIFY_RETRIES]
      --notify.interval=      Interval between attempts (default: 12s) [$NOTIFY_INTERVAL]
      --notify.method=        HTTP method (default: POST) [$NOTIFY_METHOD]
      --notify.timeout=       Request timeout (default: 30s) [$NOTIFY_TIMEOUT]
      --notify.authorization= Authorization header value [$NOTIFY_AUTHORIZATION]
      --notify.gotify=        Use Gotify format [$NOTIFY_GOTIFY]
      --notify.onlyfailures=  Only send notifications for failed jobs [$NOTIFY_ONLY_FAILURES]

Help Options:
  -h, --help                  Show this help message
```

## Notifications

Scheduler will send notifications after each job if `NOTIFY_URL` env variable or `--notify.url` flag set. Each
notification is a simple HTTP request.
HTTP method, attempts number, and interval between attempts can be configured.
Authorization via `Authorization` header also supported.
If `NOTIFY_GOTIFY` is set the gotify notification format is used.
When `NOTIFY_ONLY_FAILURES` is set, notifications will be sent only when the job fails.

Scheduler will stop retries if at least one of the following criteria met:

- reached maximum number of attempts
- server returned any `2xx` code (ex: `200`, `201`, ...)

Outgoing custom headers:

- `Content-Type: application/json`
- `User-Agent: scheduler/<version>`, where `<version>` is build version
- `Authorization: <value>` (if set)

Normal Payload:

```json
{
  "project": "compose-project",
  "service": "web",
  "container": "deadbeaf1234",
  "schedule": "@daily",
  "started": "2023-01-20T11:10:39.44006+08:00",
  "finished": "2023-01-20T11:10:39.751879+08:00",
  "failed": true,
  "error": "exit code 1"
}
```

> Note: the field `error` exists only if `failed == true`

Gotify Payload:

Using the same example as above:

```json
{
  "message": "Project: compose-project\nService: web\nContainer: deadbeaf1234\nError: status code: exit code 1, logs: 2024-12-23T18:25:23.115935476Z 2024/12/23 18:25:23 Error \nStarted: 2006-01-02T15:04:05Z07:00\nFinished: 2006-01-02T15:04:05Z07:00\nSchedule: @daily",
  "title": "Service web failed"
}
```
