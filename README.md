# envi-daq

Host software for pulling data off environmental loggers over Wi-Fi.

The loggers record temperature, humidity, pressure and battery voltage to an SD
card, one CSV file per recording session. `envi-daq` connects to each logger,
downloads any finished sessions it doesn't have yet, checks every file, and
keeps a log of every transfer attempt in SQLite.

The Wi-Fi out in the field is bad, so most of the work here is about handling
that: transfers resume where they dropped instead of starting over, corrupted
files get re-downloaded, and nothing gets saved under its final name until it
passes all the checks.

![dashboard](docs/dashboard.png)

Python 3.11+, no dependencies outside the standard library.

## Try it

You don't need any hardware. There's a simulator that acts like a logger and
can be told to drop connections, flip bytes, or return errors.

```
python -m envi_daq demo --open
```

That starts three fake loggers (good, weak and very weak signal) plus one that's
turned off, and opens the dashboard at http://127.0.0.1:8800. Hit **Sync all
loggers**. Every 45 seconds one of the loggers finishes a new session, so
there's always something to pull.

Or from the command line:

```
pip install .
envi-daq simulate --drop-rate 0.35 --corrupt-rate 0.1 &
envi-daq init
envi-daq sync -v
envi-daq history
```

```
$ envi-daq history
transfer_id  logger_id  session_id  attempt  outcome  resumed_from  bytes    error
4            ENV-0A17   S0002       2        ok       69792         21.8 KB
3            ENV-0A17   S0002       1        retry                  68.2 KB  short transfer: stream ended at byte 69792/92092
2            ENV-0A17   S0001       2        ok                     89.9 KB
1            ENV-0A17   S0001       1        retry                  89.9 KB  sha256 mismatch (got 0b940f9d5a69..., expected 954d9fd7fe97...)
```

S0002 dropped at byte 69792, and the second attempt picked up from there and
only pulled the last 21.8 KB. S0001 arrived corrupted, so it was thrown out and
downloaded again.

## How it works

Each logger serves a small HTTP API: `/api/info`, `/api/sessions` (the list of
sessions with their size and SHA-256), and `/api/sessions/<id>/data` for the
file itself. The sync for each logger goes like this:

1. Get the session list. Skip anything still recording.
2. Download each new session into `<id>.csv.part`. If a `.part` file is already
   there from a dropped attempt, ask for the rest with a `Range` header.
3. Check the size, then the SHA-256, then the CSV contents (header, column
   counts, timestamps that don't go backwards, numbers that parse).
4. If everything passes, rename it to `<id>.csv`.
5. On a timeout, dropped connection or 503, wait and try again (exponential
   backoff with jitter, 5 tries by default).

Every attempt gets a row in the `transfers` table, so you can always see how
many tries a file took and why each one failed. There's also a `link_samples`
table with the signal strength (RSSI) and latency each time the host talked to
a logger.

A few rules I made sure of:

- A file that passed the checksum but has bad contents is moved to
  `<id>.invalid.csv` and not retried, since downloading it again would just
  give the same bytes.
- If a logger starts reporting a different checksum for a session that's
  already archived, it gets flagged as a conflict and the archived copy is left
  alone.
- Only one sync can run at a time (cron, the CLI and the dashboard all share a
  lock), and a sync that gets killed halfway resumes cleanly next time.

More detail is in [docs/architecture.md](docs/architecture.md).

## Commands

| | |
|---|---|
| `init` | write a starter config.toml |
| `status` | check each logger and show what's waiting |
| `sync` | download and verify everything new |
| `sessions` | list sessions and where they're saved |
| `history` | the transfer log, one row per attempt |
| `retry` | re-queue failed sessions and sync them |
| `verify` | re-check archived files against their checksums |
| `export` | dump sessions or transfers to CSV |
| `dashboard` | the web dashboard |
| `demo` | simulated loggers + dashboard |
| `simulate` | run one fake logger |

Most commands take `--json`. Exit codes are meant for cron: 0 ok, 1 something
failed, 2 bad config, 3 a logger was unreachable, 4 a sync was already running.

## Dashboard

`envi-daq dashboard` runs a small web server on the host. The chart at the top
places each logger by signal strength, so one that's further out has a weaker
link. Below that, each logger has a card with its signal history, how many of
its transfers succeeded, and throughput. You can start a sync from the browser
and watch the progress, including where a download dropped and resumed.

It's plain HTML and JavaScript served by `http.server`, no build step.

## Tests

```
python -m unittest discover -s tests
```

The tests run against the simulator over real HTTP: dropped connections,
corruption, firmware without Range support, retries running out, conflicts,
the CLI, and the dashboard API.

## Docs

- [docs/architecture.md](docs/architecture.md): logger, Wi-Fi and host design, the API, the database
- [docs/setup.md](docs/setup.md): installing, config, connecting to loggers, running it on a schedule
- [docs/troubleshooting.md](docs/troubleshooting.md): what the error messages mean and what to do about them
