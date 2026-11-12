# Phase 4 - Incident Response Submission

**Team ID:** Not supplied; complete before submitting.

**Team name:** Not supplied; complete before submitting.

**Team members:** Not supplied; complete before submitting (2–3 members).

**Team repository:** https://github.com/harsha-lv/phase-4

**Incident ID:** RR-INC-05

## 1. Incident summary

The Inventory API experienced elevated HTTP 503 errors and response latency
during 2026-11-11 14:30–14:45 UTC. Error rate peaked at 36.2%, p95 latency
reached 5,100 ms, and all 10 database connections remained active. Inventory
requests were rejected when the pool could no longer provide connections.

## 2. Root cause

In `app/inventory.py`, `lookup_inventory()` acquired a connection before the
query, but released it only on the successful path. Raising `DatabaseTimeout`
skipped `pool.release(connection)`, leaking one connection per failed query.
Repeated timeouts exhausted the shared pool and prevented later requests from
acquiring connections, sustaining failures even after the original queries ended.

Commit `d927c0a3be61b27ce3bc987a1233c450bc03e3d9` removed the `try/finally`
cleanup present in baseline `f113535`. This identifies the code regression,
not a deployment during the incident: the alert context explicitly records no
deployment in the incident window. The evidence does not establish what caused
the initial database timeouts; the confirmed defect is the failure to release
connections when they occur.

## 3. Evidence

1. **Application logs (`incident/application.log`):** `req-601` acquires at
   `14:30:02Z` with `active=8 max=10`, then reports `db_query_timeout` at
   `14:30:06Z` and HTTP 503, without a corresponding release. `req-602` and
   `req-603` similarly acquire at active counts 9 and 10, time out, and do not
   release. At `14:30:17Z`, `req-604` reports
   `event=pool_acquire_rejected reason=pool_exhausted active=10 max=10`.
   Rejections persist at `14:35:03Z` and `14:40:08Z`. Earlier successful requests
   `req-501` and `req-502` both have explicit `pool_release` events.
2. **Historical metrics (`incident/metrics.csv`):** at `14:25:00Z`, traffic is
   134 requests/minute, errors 1.2%, p95 175 ms, pool 4/10, CPU 28%. At
   `14:30:00Z`, errors rise to 18.4%, p95 to 2,100 ms, and pool usage to 8/10.
   By `14:35:00Z` the pool is 10/10; at `14:40:00Z`, errors are 36.2%, p95 is
   5,100 ms, traffic is 141/minute, and CPU is 31%. The sharp degradation tracks
   pool saturation, not a comparable traffic surge or CPU saturation.
3. **Alerts (`incident/alerts.json`):** the critical incident records error-rate,
   latency, and pool-saturation alerts, with observed values 36.2%, 5,100 ms,
   and 10/10 (100%). Context says traffic is stable near 140/minute, CPU is
   approximately 30%, and no deployment is recorded during the window.
   Its `fired_at` is `14:31:20Z`; the snapshot's observed values match later
   metrics, so they are not treated as exact measurements at the firing time.
4. **Dashboard (`incident/dashboard.html`):** the static production summary
   shows 36.2% errors, 5,100 ms p95, pool 10/10, and 31% CPU, consistent with
   the 14:40 metric sample. This is supplied incident evidence, not live
   post-fix production telemetry.

## 4. Corrective action

Restored `try/finally` around the query after successful acquisition. The
`finally` block releases the connection exactly once whether the query returns
stock or raises an exception. `DatabaseTimeout` still propagates; it is not
swallowed. Known-SKU quantities, unknown-SKU zero results, pool capacity checks,
and rejection of exhausted acquisitions remain unchanged. Acquisition remains
outside `try`, so an unsuccessful acquire does not attempt to release a
connection that was never acquired.

No tests, alert thresholds, incident artifacts, pool limits, or workflow checks
were weakened. The fix prevents new leaks; it does not claim to repair already
leaked connections in a running production process. Any real rollout would need
to replace affected processes and observe fresh telemetry.

## 5. Validation

Before the fix, `python3 -m unittest discover -s tests -v` ran six tests with
three failures, and the verifier reported one connection still active after
the first timeout. After the fix, on local Python 3.14.10:

```text
$ python3 -m unittest discover -s tests -v
test_database_timeout_releases_connection ... ok
test_known_sku_returns_stock ... ok
test_pool_does_not_exhaust_after_repeated_timeouts ... ok
test_service_recovers_after_database_timeout ... ok
test_successful_request_releases_connection ... ok
test_unknown_sku_returns_zero ... ok
Ran 6 tests in 0.000s
OK

$ python3 verify_recovery.py
RECOVERY VERIFIED
Repeated database timeouts did not leak connections.
A normal inventory request succeeded after the timeouts.
FLAG{POOL_RELEASE_RECOVERED}
```

The verifier checks zero active connections after each of three simulated
timeouts and then a normal lookup returning 25 with zero active connections.
`git diff --check` also passes. This demonstrates recovery in the supplied
application model, not live production recovery.

**Final commit hash:** Run `git rev-parse HEAD` on the final submission checkout.

**GitHub Actions:** https://github.com/harsha-lv/phase-4/actions

**Successful run URL:** Pending verification of the final pushed commit.
The unauthenticated GitHub API was rate-limited during investigation; local
success must not be represented as a verified hosted workflow result.

## 6. Final flag

`FLAG{POOL_RELEASE_RECOVERED}`
