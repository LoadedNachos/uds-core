# Local CI performance experiments

## Agent handoff

This document is the working log for improving UDS Core pull request CI through test-code and test-fixture changes. Focus on test execution, polling, fixture deployment, and reliability. GitHub workflow ordering was investigated separately and is out of scope for the remaining work.

### Working locations

- Experiment checkout: `/home/chance/Personal/uds-core`
- This document: `/home/chance/Personal/uds-core/docs/dev/ci-performance-experiments.md`
- Historical workflow reference: `/home/chance/Unicorn/uds-core/docs/dev/ci-performance-experiments.md`
- Experiment branch at the start: `ci-testing`, commit `66009331` (`WIP CI optimization change`)
- No changes were pushed or published.

### Safety and current state

- The pre-existing WIP in `.github/actions/values-tests/action.yaml`, `.github/workflows/pull-request-conditionals.yaml`, `.github/workflows/test.yaml`, `tasks.yaml`, and `tasks/test.yaml` was removed at the user's request. Do not restore it unless explicitly asked.
- The checkout is intentionally a clean slate. All experiment source patches were reverted. Only this note should remain as the local investigation artifact.
- Proven changes are documented as candidates, not implemented changes. Reapply and validate them one at a time before accepting them.
- Falco integration concurrency and Envoy asynchronous cleanup were tested but are not retained.
- The disposable k3d cluster named `uds` was deleted after testing. Recreate it before running cluster-backed tests.
- No dependencies were added. Existing `node_modules` came from the repository's normal `npm ci` setup.

### Reproduction setup

Run Vitest commands from `/home/chance/Personal/uds-core/test/vitest`:

```sh
uds run -f tasks/setup.yaml k3d-test-cluster --no-progress
uds zarf package deploy build/zarf-package-core-amd64-1.12.0.tar.zst --confirm --components '*' \
  --set INSECURE_ADMIN_PASSWORD_GENERATION=true \
  --set FALCO_SANDBOX_RULES_ENABLED=true \
  --set FALCO_INCUBATING_RULES_ENABLED=true
uds run -f src/test/tasks.yaml create-deploy --no-progress
time npx vitest run <file>.spec.ts
```

The measured environment used UDS CLI `v0.36.0`, k3d `v5.7.4`, Node `22`, Vitest `v4.1.11`, and Playwright `v1.63.0`. The Core package deployment took about 11.7 minutes, so it is infrastructure setup time and should not be mixed with test-code timing.

Before running Keycloak-dependent tests, create the test user used by Playwright and several Vitest suites:

```sh
uds run -f tasks/test.yaml common-setup:keycloak-user --no-progress --with group=/UDS\ Core/Admin
```

The cached package used during this investigation was `build/zarf-package-core-amd64-1.12.0.tar.zst`. If it is unavailable, use the repository's normal Core package build task before deployment. The test fixture task creates and deploys the test-app package, including the Loki ruler ConfigMap, network test apps, UDP resources, and podinfo.

### Exact experiment recipes

Run each baseline and variant from a clean source state. The variant locations below describe the temporary patches used during the investigation.

| Candidate | Baseline and variant | Code location | Evidence |
| --- | --- | --- | --- |
| Grafana parallelism | Baseline: run `npx playwright test grafana.test.ts --workers=9 --retries=0` with `test.describe.configure({ mode: "serial" })`. Variant: remove that statement. | `test/playwright/grafana.test.ts`, top-level describe configuration | Local Docker Playwright run: 54s to 13s, 9 tests passed. |
| Uptime concurrency | Baseline: `time npx vitest run uptime.spec.ts`. Variant: change both top-level `describe("Uptime ...")` blocks to `describe.concurrent(...)`. | `test/vitest/uptime.spec.ts`, the `Uptime Probes` and `Uptime Recording Rules` blocks | Local fixture-backed run: 29s to 5s, all 41 tests passed. |
| Falco temporary pod image policy | Baseline: `time npx vitest run falco.spec.ts` with the default `alpine:latest` pod image policy. Variant: set `imagePullPolicy: "IfNotPresent"` in `createTempPod`. Import a local image with `k3d image import alpine:latest -c uds` when testing offline behavior. | `test/vitest/helpers/k8s.ts`, `createTempPod` container spec | Local run: 31s `ImagePullBackOff` failure to 3.7s passing run. |
| Failed temporary-pod cleanup | Baseline: allow `createTempPod` readiness to throw. Variant: wrap `waitForPodReady` in `try/catch`, call `deleteTempPod` on failure, then rethrow. | `test/vitest/helpers/k8s.ts`, `createTempPod` after `createNamespacedPod` | The baseline leaked failed Falco pods. The cleanup variant was exercised, but final pod-list verification was interrupted. |
| Egress waypoint readiness | Baseline: `time EGRESS_TESTS=false npx vitest run network.spec.ts -t 'Egress Ambient'` with the fixed 10-second sleep. Variant: call `waitForPodReady` for label `gateway.networking.k8s.io/gateway-name=egress-waypoint` in namespace `istio-egress-ambient`. | `test/vitest/network.spec.ts`, `Network Policy Validation` and `Egress Ambient` test | Local failure-path run: about 47s to 6.5s. Successful external-egress path was unavailable locally. |
| Envoy polling interval | Baseline: `time npx vitest run envoy-gateway.spec.ts` with 5-second poll intervals. Variant: use 1-second intervals for Gateway, Service, and Pod readiness polls. Skip the cleanup wait only when measuring the test body. | `test/vitest/envoy-gateway.spec.ts`, `pollUntilSuccess` calls in the test and `afterAll` cleanup | Local test body: about 17s to 13s. |
| Envoy asynchronous cleanup | Baseline: strict `afterAll` namespace deletion wait. Variant: use a unique test namespace and return after issuing deletion on disposable CI. | `test/vitest/envoy-gateway.spec.ts`, `TEST_NAMESPACE`, `afterAll`, and `waitForNamespaceDeleted` | Local run: about 193s to 17s. Needs careful CI validation before acceptance. |
| Falco integration concurrency | Baseline: `time npx vitest run falco-integration.spec.ts` with a normal top-level `describe`. Variant: use `describe.concurrent` for the integration block. | `test/vitest/falco-integration.spec.ts`, top-level `Falco Integration e2e Tests` block | Local failure-path run: about 257s to 125s. Two events were not delivered, so this is not accepted. |
| HTTP request timeouts | No valid baseline/variant run was completed. Add bounded `fetch` requests and compare broken-forward behavior. | `test/vitest/helpers/prometheus.ts`, `test/vitest/helpers/alertmanager.ts`, `test/vitest/loki.spec.ts`, and related query helpers | Inferred from requests that can outlive `pollUntilSuccess`; not benchmarked. |

For the Playwright recipe, use the same Docker invocation as `tasks/test.yaml`, with `npm ci` and `npx playwright test` inside `mcr.microsoft.com/playwright:v1.63.0-noble`. Do not compare a run that includes the initial multi-minute browser-image pull with a cached-image run.

### Failure inventory from the full local run

The full command was `EGRESS_TESTS=false UDP_EXPOSE_TESTS=false npx vitest run --reporter=verbose` after deploying test fixtures. It took about 192 seconds elapsed and produced these failures:

- `falco.spec.ts` and all three tests in `falco-integration.spec.ts` initially hit 30-second readiness waits because `alpine:latest` used an image-pull policy that attempted Docker Hub access. After using `IfNotPresent`, the basic Falco test passed in 3.7 seconds, but two Sidekick event tests still timed out after 120 seconds because the local Falco event path did not deliver those events.
- `network.spec.ts` failed the Google/Bing success checks with `HTTP_CODE:000`. The local k3d cluster could not reach the external sites. These are environment failures, not evidence that the network test logic is wrong.
- `keycloak-notifications.spec.ts` lacked the `keycloak-admin-password` secret because complete Keycloak setup had not been applied.
- `keycloak-fleet-admin.spec.ts` later timed out waiting for its test pod, so its result was not a valid performance measurement.
- Running `uptime.spec.ts` before deploying test fixtures left several expected endpoints absent and caused long polling behavior. Run `src/test/tasks.yaml create-deploy` first.

### Cleanup after experiments

The investigation cluster has already been deleted. If a future run is interrupted, remove only resources created by these tests:

```sh
kubectl delete pods -A -l created-by=uds-core-vitest --ignore-not-found
k3d cluster delete uds
```

Check for namespaces created by the Envoy test and remove them only after confirming they belong to the run. The Envoy namespace deletion can take several minutes on a busy cluster.

### How to continue

1. Inspect `git status` and the current diffs before changing anything.
2. Reapply one documented candidate at a time and verify the relevant tests, especially the successful Egress path and Falco event delivery.
3. Treat failure-path timing as reliability evidence, not proof of successful-path speedup.
4. Re-test deferred Falco concurrency and Envoy cleanup before accepting them.
5. Record new measurements, pass/fail counts, environment conditions, and implementation status in this document.

### Call to action for the next agent

Use the documented results as starting hypotheses, not as an exhaustive list. Inspect the current Vitest, Playwright, polling, Kubernetes helper, and test-fixture code for additional gains. For each new idea:

1. Establish a baseline on the same fixture and environment.
2. Change one behavior at a time in a disposable worktree or patch.
3. Run the affected test repeatedly on both success and failure paths.
4. Check correctness, cleanup, flake rate, and total elapsed time.
5. Keep only evidence-backed improvements and record the result here.

## Success goal

The only acceptance goal is a test suite that is faster and more reliable. Do not accept a speed improvement that increases flakes, leaks resources, weakens assertions, or depends on a fragile environment assumption. Do not require a fixed percentage or time threshold; use the best measured improvement that preserves reliable test behavior.

## Validated improvements

| Opportunity | Local result | Assessment |
| --- | ---: | --- |
| Envoy namespace cleanup | About 193s to 17s | Highest-value gain. Use unique namespaces and asynchronous cleanup on disposable CI clusters. |
| Grafana Playwright serial mode | 54s to 13s | High-confidence gain. Nine independent tests passed in parallel. |
| Uptime suites | 29s to 5s | High-confidence gain. All 41 tests passed concurrently. |
| Falco temporary pod image policy | 31s failure to 3.7s pass | Use `IfNotPresent` to avoid registry waits and improve offline reliability. |
| Egress waypoint readiness | 47s failure path to 6.5s | Replace the fixed 10-second sleep with an active readiness check. Success path still needs validation. |
| Envoy polling interval | About 17s to 13s | Small gain from reducing 5-second polling to 1 second. |

## Needs more validation

- Falco integration concurrency reduced a failing run from about 257s to 125s. Two events were not delivered locally, so successful-path validation is still required.
- Prometheus, Alertmanager, and Loki fetch helpers have no request timeout. Add bounded requests so broken forwards fail promptly.
- Failed temporary-pod readiness leaked `ImagePullBackOff` pods. Cleanup-on-failure was tested, but final verification was interrupted.

## Implementation status

No performance code changes are retained locally. The checkout is intentionally a clean slate with this note as the investigation artifact.

Validated candidates ready for reapplication and confirmation:

- Grafana tests run in parallel.
- Uptime probe and recording-rule suites run concurrently.
- Temporary Falco pods use `IfNotPresent` and clean up after readiness failures.
- Egress tests wait for waypoint readiness instead of using a fixed sleep.

Not yet accepted:

- Falco integration test concurrency.
- Envoy asynchronous cleanup and unique-namespace changes.

## Limitations

- Local k3d egress to external sites such as Google and Bing was blocked, so network egress results included environment failures.
- The local cluster lacked complete Keycloak setup state, so Keycloak-related full-suite results were not representative.
- Several timing improvements were measured on failure paths. Their successful-path impact still needs CI or a correctly configured local-cluster validation.

## Validation source

| Finding | Validation source |
| --- | --- |
| Workflow split and artifact reuse | GitHub CI runs documented above |
| Grafana parallelism | Local k3d-backed Playwright run |
| Uptime concurrency | Local k3d-backed Vitest run, all 41 tests passed |
| Falco image policy and cleanup | Local k3d-backed Vitest runs |
| Egress waypoint readiness | Local k3d failure-path run |
| Envoy polling interval | Local k3d-backed Vitest run |
| Falco integration concurrency | Local k3d failure-path run only |
| Request timeouts for HTTP helpers | Inferred from helper behavior, not benchmarked |
| Optional fixture removal and Ambient serial removal | Local k3d-backed experiments |
| Keycloak concurrency | Invalid local comparison because alerts were already active |

## Did not justify a change

- Removing optional egress fixtures did not improve deployment time.
- Removing Ambient Waypoint Playwright serial mode saved only about 1 second.
- Keycloak concurrency appeared faster only because alerts were already active. The result was invalid.
- Playwright was already near its parallelism limit. Loki, Prometheus, UDP, and most policy tests were already short.

## Environment findings

The full local Vitest pass used `EGRESS_TESTS=false UDP_EXPOSE_TESTS=false`, took about 192 seconds elapsed, and ran 22 files with 120 passing tests, 7 failing tests, and 14 skipped tests. Failures were caused by blocked external egress in k3d, missing Keycloak setup state, and default `alpine:latest` image pulls. Test-fixture creation and deployment took about 2 minutes 8 seconds. Core package deployment took about 11.7 minutes and is infrastructure time, not test execution time.

## Recommended order

1. Remove Grafana serial mode.
2. Make Envoy cleanup asynchronous for disposable CI runs.
3. Run Uptime suites concurrently.
4. Set temporary test pods to `IfNotPresent` and clean up failed readiness attempts.
5. Replace fixed waypoint sleeps and add request timeouts.
6. Re-test Falco concurrency with successful event delivery.
