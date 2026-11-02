# Trigger Kometa from Radarr & Sonarr on Unraid

## Overview

When Radarr or Sonarr finishes an import, Kometa runs for just that library in a limited mode (default `diff`) after a short delay, so new movies and episodes get their overlays and collections within minutes instead of waiting for the next scheduled run.

How it works:

1. Radarr or Sonarr sends a webhook to **webhookd**, a small webhook server running on the Unraid host (not in a container).
2. webhookd checks the username and password, then runs the matching script: `kometa-radarr.sh` or `kometa-sonarr.sh`.
3. The script answers straight away and starts a background job that waits a configurable delay, so Plex has time to scan the new files.
4. If a webhook-triggered Kometa run is already busy, the job waits for it to finish (up to 1 hour).
5. The job runs `docker exec` on the Kometa container: `python kometa.py --run-libraries <library> --mode <mode>`.

The setup is safe because no container gets access to the Docker socket. Radarr and Sonarr can only send an HTTP request; the host decides which single command it runs.

Imports that arrive in a burst are merged: if a run is already queued for a library, new webhooks for that library are ignored, and the queued run covers them all.

## Requirements

- Unraid with the **Community Applications** plugin.
- **Kometa** running as a Docker container, with a working `config.yml` and at least one completed full run. The first `diff` run without a previous snapshot processes the whole library.
- **Radarr** and/or **Sonarr** able to reach the Unraid host's IP address.
- Optional: the **User Scripts** plugin, used to start webhookd automatically after a reboot.
- Terminal access to Unraid (web terminal or SSH).

## Step 1: Install webhookd

1. In Unraid, open **Apps** and search for **webhookd** (plugin by fejich). Install it.
2. The plugin does not create its config folder until the service starts for the first time. Start it once, then stop it again:

```bash
/etc/rc.d/rc.webhookd start
/etc/rc.d/rc.webhookd stop
```

The start output shows the port it listens on, for example `addr=:8090`. Note this port; you need it later.

You should now have:

- `/mnt/user/appdata/webhookd/webhookd.env`
- `/mnt/user/appdata/webhookd/scripts/`

If `rc.webhookd` is not found, the plugin did not install correctly. Check with `ls -l /etc/rc.d/rc.webhookd /usr/bin/webhookd` and reinstall if either is missing.

## Step 2: Configure webhookd

Edit `/mnt/user/appdata/webhookd/webhookd.env` and make sure it contains these lines. Keep the port from Step 1, or pick a free one:

```bash
WHD_LISTEN_ADDR=:8090
WHD_HOOK_SCRIPTS=/mnt/user/appdata/webhookd/scripts
WHD_PASSWD_FILE=/mnt/user/appdata/webhookd/.htpasswd
```

Create the password file. Unraid has no `htpasswd` tool and webhookd needs bcrypt hashes, so use a throwaway container. Replace `YOUR_PASSWORD` with a password of your own:

```bash
docker run --rm httpd:alpine htpasswd -nbB kometa 'YOUR_PASSWORD' \
  > /mnt/user/appdata/webhookd/.htpasswd
```

Check it: `cat /mnt/user/appdata/webhookd/.htpasswd` should show one line starting with `kometa:$2y$`.

The username is `kometa`. You enter the same username and password in Radarr and Sonarr later.

## Step 3: Check your Kometa container

Two values in the scripts depend on your Kometa image, so check them before you go on.

**Container name.** Docker names are case-sensitive; Unraid templates often use `Kometa`:

```bash
docker ps --format '{{.Names}}' | grep -i kometa
```

**Location of `kometa.py`.** This becomes the working directory (`-w`) for `docker exec`:

```bash
docker exec Kometa sh -c 'find / -name kometa.py -not -path "*/proc/*" 2>/dev/null'
```

If it prints `/kometa.py`, the working directory is `/`. If it prints `/app/kometa/kometa.py`, use `/app/kometa`.

**Library names.** Note the exact library names from your Kometa `config.yml`, for example `Movies` and `TV Shows`.

Confirm it all works by running Kometa by hand. Use your container name, working directory and library name:

```bash
docker exec -w / Kometa python kometa.py --run-libraries "Movies" --mode diff
```

If this runs, the scripts will work too.

## Step 4: Add the scripts

Create two files in `/mnt/user/appdata/webhookd/scripts/`. The file name becomes the webhook URL: `kometa-radarr.sh` is reached at `/kometa-radarr`. All settings sit at the top; adjust them to the values from Step 3.

**`kometa-radarr.sh`**

```bash
#!/bin/bash
# ---------------------------------------------------------------
# Settings
# ---------------------------------------------------------------
LIB="Movies"         # Library name exactly as in Kometa's config.yml
DELAY=120            # Seconds to wait so Plex can pick up the new files
CONTAINER=Kometa     # Kometa container name (case-sensitive)
WORKDIR="/"          # Folder inside the container that holds kometa.py
LOG="/tmp/kometa-webhook-radarr.log"

# Run mode, see https://kometa.wiki/en/latest/kometa/guides/scheduling/#choose-a-mode
#   full          - every item in the library
#   added(1)      - items added in the last day (also 6h, 30m, 2d)
#   diff          - items not in Kometa's previous library snapshot
#   index(A-F#)   - titles starting A-F, # = numbers/symbols
#   items(ids)    - specific items, e.g. items(tmdb:550|imdb:tt0133093)
MODE="diff"

# Optional: limit which tasks run (leave empty for all tasks)
#   -ov = overlays only, -co = collections only, -mo = metadata only
ONLY=""
# ---------------------------------------------------------------

PENDING="/tmp/kometa-pending-${LIB// /_}"

# Let the Test button pass without running Kometa
echo "$1" | grep -q '"eventType": *"Test"' && { echo "Test OK"; exit 0; }

# A delayed run is already queued for this library and will cover this import too
if [[ -e "$PENDING" ]]; then
  echo "Run already pending for $LIB"
  exit 0
fi
touch "$PENDING"

# Make the settings available to the background job
export LIB DELAY CONTAINER WORKDIR MODE ONLY PENDING

# Detach so webhookd can answer immediately
setsid nohup bash -c '
  sleep "$DELAY"
  WAITED=0
  while docker top "$CONTAINER" aux | grep -q "run-libraries"; do
    (( WAITED >= 3600 )) && { echo "$(date) gave up waiting"; rm -f "$PENDING"; exit 1; }
    sleep 60; (( WAITED += 60 ))
  done
  rm -f "$PENDING"
  echo "$(date) starting Kometa: $LIB mode=$MODE ${ONLY:-all tasks}"
  docker exec -d -w "$WORKDIR" "$CONTAINER" python kometa.py --run-libraries "$LIB" --mode "$MODE" ${ONLY:+"$ONLY"}
' >"$LOG" 2>&1 &

echo "Kometa queued for $LIB in ${DELAY}s (mode: $MODE${ONLY:+, $ONLY})"
```

**`kometa-sonarr.sh`**

Copy `kometa-radarr.sh` and change only these three settings:

```bash
LIB="TV Shows"
DELAY=150
LOG="/tmp/kometa-webhook-sonarr.log"
```

Sonarr gets a longer delay because a download can hold many episodes. Keep the two delays at least 30 seconds apart, so that if both fire together the second run sees the first and waits.

Make both scripts executable:

```bash
chmod +x /mnt/user/appdata/webhookd/scripts/*.sh
```

Then start webhookd:

```bash
/etc/rc.d/rc.webhookd start
```

## Step 5: Test from the terminal

Test before touching Radarr or Sonarr. Use your own password and port:

```bash
curl -u kometa:YOUR_PASSWORD -X POST http://localhost:8090/kometa-radarr \
  -d '{"eventType":"Download"}'
```

It should print `Kometa queued for Movies in 120s (mode: diff)`.

While the delay runs, the job is waiting:

```bash
ls -l /tmp/kometa-pending-*
ps aux | grep "[r]un-libraries"
```

After the delay, Kometa should be running for the library and the pending file should be gone:

```bash
docker top Kometa aux | grep run-libraries
cat /tmp/kometa-webhook-radarr.log
```

Follow the run in Kometa's own log. `docker logs` does not show it, because a `docker exec` process is not the container's main process. Adjust the path to your Kometa appdata folder:

```bash
tail -f /mnt/user/appdata/kometa/logs/meta.log
```

## Step 6: Connect Radarr and Sonarr

In Radarr, go to **Settings → Connect → + → Webhook** and fill in:

| Field | Radarr | Sonarr |
| --- | --- | --- |
| Webhook URL | `http://<unraid-ip>:8090/kometa-radarr` | `http://<unraid-ip>:8090/kometa-sonarr` |
| Method | POST | POST |
| Username | `kometa` | `kometa` |
| Password | your password | your password |
| Triggers | On File Import, On File Upgrade | On Import Complete, On File Upgrade |

Do the same in Sonarr with its own URL. Click **Test**, then **Save**. The test passes without starting Kometa.

Trigger choices:

- **Sonarr: On Import Complete** fires once after a whole download is imported, rather than once per episode with On File Import. A season pack then gives one Kometa run.
- **On File Upgrade** is only useful if you use overlays based on file properties (resolution, HDR, audio codec, edition). An upgrade changes those; collections are not affected.
- Leave **On Grab** and **On Rename** off. Every enabled trigger except Test starts Kometa.

A 401 error on Test almost always means the Username field is empty or the password does not match the `.htpasswd` file.

## Step 7: Start webhookd after a reboot

webhookd does not reliably start by itself after a reboot, and its config lives on the array, so start it once the array is up.

1. Go to **Settings → User Scripts → Add New Script** and name it `webhookd-start`.
2. Set the script content to:

```bash
#!/bin/bash
/etc/rc.d/rc.webhookd restart
```

3. Set the schedule to **At Startup of Array**.

`restart` works whether or not the plugin already started the service. After the next reboot, check it is running with `ps aux | grep [w]ebhookd`.

## Settings reference

All settings are at the top of each script.

| Setting | Default | What it does |
| --- | --- | --- |
| `LIB` | `Movies` / `TV Shows` | Library name, exactly as in Kometa's `config.yml` |
| `DELAY` | `120` / `150` | Seconds to wait before running, so Plex can scan the new files |
| `CONTAINER` | `Kometa` | Kometa container name, case-sensitive |
| `WORKDIR` | `/` | Folder inside the container that holds `kometa.py` |
| `LOG` | `/tmp/kometa-webhook-<app>.log` | Output of the last background job; overwritten each run |
| `MODE` | `diff` | Which items Kometa processes (see below) |
| `ONLY` | empty | Limit to one task type: `-ov` overlays, `-co` collections, `-mo` metadata |

`MODE` options, from the [Kometa scheduling guide](https://kometa.wiki/en/latest/kometa/guides/scheduling/#choose-a-mode):

| Mode | Processes |
| --- | --- |
| `full` | Every item in the library |
| `added(1)` | Items added in the last day; also `6h`, `30m`, `2d` |
| `diff` | Items not in Kometa's previous library snapshot |
| `index(A-F#)` | Titles starting A to F; `#` adds numbers and symbols |
| `items(tmdb:550\|imdb:tt0133093)` | Specific items by ID |

`diff` and `added(…)` suit webhook runs. These limited modes do not remove stale collection members or reorder whole collections, so keep a scheduled `full` run in Kometa for cleanup.

If Plex only finds new files through its own scheduled scans, set `DELAY` longer than that scan interval. With Radarr and Sonarr notifying Plex directly, 2 minutes is usually enough.

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| No `/mnt/user/appdata/webhookd/` folder | The service has never started | Run `/etc/rc.d/rc.webhookd start` once |
| 401 Unauthorized on Test | Username empty or password wrong | Fill in both fields; regenerate `.htpasswd` if needed; check `WHD_PASSWD_FILE` and restart webhookd |
| curl prints `Run already pending` but nothing runs | Leftover pending file from an earlier failed test | `rm -f /tmp/kometa-pending-*` |
| Nothing in `docker logs` | `docker exec` output never appears there | Read Kometa's `meta.log` instead |
| Log shows `No such container` | `CONTAINER` name wrong or wrong case | Check with `docker ps --format '{{.Names}}'` |
| Log shows `chdir to cwd … failed` | `WORKDIR` wrong for your Kometa image | Find `kometa.py` (Step 3) and set `WORKDIR` |
| Kometa does not run after a reboot | webhookd not started | Add the User Script from Step 7 |

The first place to look is always the script's own log, for example `cat /tmp/kometa-webhook-radarr.log`. It holds the output of the last background job, including any error from `docker exec`.

One known limitation: the script only detects Kometa runs started with `--run-libraries`. Kometa's own scheduled run happens inside the container's main process, so a webhook run can overlap with it. Scheduling Kometa's full run at night, when few imports happen, avoids most overlaps.
