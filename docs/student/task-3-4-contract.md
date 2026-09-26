# Task 3.4 — SLO and alert contract

Configure the one deployed alert rule's actionable window, then run the two supplied exercise
scripts that force one exception to the dead-letter queue and prove the alert fires, then redrive
it and prove the alert resolves. You edit one duration in `infra/observability/alerts.yml`. You do
not touch the metric it reads, the queue's own dead-letter policy from Task 3.3, or how the worker
is wired.

## The one SLO and its one alert

- **SLO**: the dead-letter queue's depth is the one user-impact signal this Task assesses. A
  non-zero, sustained reading means at least one exception has been dead-lettered and not yet
  redriven — the shipment's exception workflow silently stopped making progress for that customer.
- **Alert**: `ColdlineDeadLetterQueueBacklog`, defined in `infra/observability/alerts.yml`, fires on
  `coldline_job_queue_dead_letter_depth > 0` sustained for the rule's `for` duration.
- **Metric**: `coldline_job_queue_dead_letter_depth`, a Gauge the worker's own
  `src/worker/queue_monitor.py` poller reports every two seconds, supplied and complete.

## What is assessed, and by whom

| Assessed | By |
|---|---|
| The pull request changes only `infra/observability/alerts.yml` and `submission.yaml` | Automated, in this repository |
| The deployed alert rule's `for` duration is inside the published bound | Automated, against Prometheus's own `/api/v1/rules` |
| Forcing one exception to the dead-letter queue makes the alert reach the `active` state within a bounded wait | Automated, against the running Alertmanager |
| Redriving that message makes the exception reach `COMPLETED` and the alert resolve | Automated, against the running API and Alertmanager |
| Your reasoning about why a `for` of `90s` passes Prometheus's parser but is still the wrong choice | Your instructor, at the Project Defense |

## What is already supplied

| Supplied | Where | Note |
|---|---|---|
| The dead-letter depth gauge and its poller | `src/worker/metrics.py`, `src/worker/queue_monitor.py` | conformance-tested; do not edit |
| Composition wiring to the poller | `src/worker/bootstrap.py`, `src/worker/config.py` | do not edit |
| Alertmanager and its configuration | `compose.yaml`, `infra/observability/alertmanager.yml` | do not edit; no real notification integration is configured, on purpose |
| Prometheus's rule loading and Alertmanager wiring | `infra/observability/prometheus.yml` | do not edit |
| The two failure-exercise scripts | `tests/failure/trigger_alert_load.py`, `tests/failure/verify_alert_recovery.py` | run them; do not edit them |
| Task 3.3's own settled dead-letter redrive policy | `compose.yaml` | unchanged; `compose.yaml` is no longer this Task's editable surface |

## The one setting

`infra/observability/alerts.yml` already defines `ColdlineDeadLetterQueueBacklog`, but the
starter's `for` is `90s`. That is inside the published bound (`for` in `[5s, 120s]`), so Prometheus
loads the rule fine; it is still the wrong choice. This exercise's forced failure and redrive
complete in well under a minute, so a rule that needs 90 continuous seconds of backlog before it
is allowed to fire never fires during the exercise — even though the dead-letter queue really is
backed up the whole time. Lower `for` to a value that actually fires within the exercise window
while staying inside the published bound.

How long is that window? The published bound is what Prometheus will accept; it is wider than what
this exercise can observe. `poe trigger-alert-load`, the same script the automated check runs,
waits at most **45 seconds** for the alert to reach `active` before giving up, counted from when it
restarts the worker. The rule's `for` clock only starts once the restarted worker reports the
dead-letter depth and Prometheus has scraped and evaluated it, which took about 6 to 12 seconds in
local measurements. Alertmanager's API lists the alert as `active` as soon as Prometheus fires it;
its 5-second `group_wait` delays notifications, not that state. Measured locally, only a `for` of
about 30 seconds or less reached `active` in time: `30s` fired about 37 to 39 seconds after the
restart, while `35s`, `40s`, `45s`, `60s`, and `120s` all timed out. A slower machine or CI runner
can take longer to report, so leave a margin below that. A value that is too long shows up as a
timeout, not a rule error. Choose a value that can fire inside that 45-second window.

Then reload Prometheus: editing `alerts.yml` only changes the file on disk. Docker Compose's `up`
(what `poe start` runs) only recreates a container when the container's own configuration changes
— its image, environment, build args, and so on — it does not track a bind-mounted file's content,
so running `poe start` again does **not** make Prometheus re-read `alerts.yml`; the deployed rule
silently keeps evaluating the *old* `for` value, which is confusing precisely because the file on
disk looks right. Run `poe reload-alerts` instead — it restarts just the Prometheus container, which
does make it re-read the file — then run `poe slo-contract` until it passes.

## The two exercises

Run these against the live stack, in order, after `poe start`:

```shell
poe worker-stop           # trigger-alert-load forces one exception to the dead-letter queue
poe trigger-alert-load    # forces the failure, restarts the worker, and waits for the alert to fire
poe verify-alert-recovery # redrives the message and waits for the exception and the alert to recover
```

`poe trigger-alert-load` prints the exception id, the dead-letter queue depth, and the alert's
state, which must reach `active`. `poe verify-alert-recovery` prints the same exception id's final
state, which must be `COMPLETED`, and the alert's final state, which must be `resolved` or absent.

## Commands

```shell
poe slo-contract   # the automated bound, firing, and resolution checks
poe verify         # the full public student verification path
```

## What the checks verify

| Check | What it looks at |
|---|---|
| `test_alert_threshold_is_actionable_and_recovers` | The deployed rule's `for` is inside `[5s, 120s]`; forcing one exception to the dead-letter queue makes the alert reach `active` inside a bounded wait of at most 45 seconds after the worker restarts, so pick a `for` that can fire in that window (about 30 seconds or less); redriving it makes the exception reach `COMPLETED` and the alert report `resolved` or absent |

## Student-editable paths

- `infra/observability/alerts.yml`
- `submission.yaml`

Keep the metrics poller, its composition wiring, Alertmanager's configuration, and every test file
exactly as supplied. The dead-letter depth signal and the alert pipeline are conformance-tested
scaffolding, not this Task's assignment; configuring the one alert's actionable window correctly,
and observing what it does, is.
