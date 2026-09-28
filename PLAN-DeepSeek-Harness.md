# PLAN — DeepSeek Harness (dsh) Olares App

> Context + runbook for every future update/upgrade of this packaging repo.
> Keep this file synchronized with reality; it is the first file to read (and update) each session.

## 1. What this repo is

This repo packages [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) (`dsh`,
open-source agent harness by DeepSeek AI, MIT) as a self-contained Olares app chart:

```
dsh/                              # the OAC (chart folder) — what gets uploaded & submitted
├── Chart.yaml                    # Helm chart identity: name dsh, version == metadata.version
├── OlaresManifest.yaml           # manifest: metadata, entrance, spec, envs, options, workloadReplicas
├── values.yaml                   # olaresEnv PROXY_* stubs, workloads.dsh.replicaCount, appData default
├── owners                        # listed owners for the public Market (beclab/apps)
├── i18n/en-US/OlaresManifest.yaml  # English locale copy
├── i18n/zh-CN/OlaresManifest.yaml  # Chinese locale copy
└── templates/
    ├── deployment.yaml           # workload: init-permissions + dsh + ui-shim sidecar
    ├── service.yaml              # dsh-svc :3080 → ui-shim :3090
    ├── secret.yaml               # dsh-proxy-auth (PROXY_USERNAME / PROXY_PASSWORD)
    └── shim-configmap.yaml       # nginx config for the compression-normalizing sidecar
assets/
├── icon/dsh-256.png              # 256x256 app icon (used by metadata.icon + entrance icon)
├── icon/source_deepseek-512.png  # upstream source (selfhst icons PNG, official #4D6BFE whale)
└── listing/1..3.png              # 1440x900 featured/promote images (deterministic Pillow renders)
```

- **Submitter/owner:** `abidals`
- **Category:** `AI`
- **Architectures:** `amd64` + `arm64` (the community image is multi-arch)
- **Installs on:** Olares ≥ 1.12.6 (`options.dependencies` olares system `>=1.12.6-0`)
- **Local proof instance:** `my@cgtale.com` (amd64 Olares, olares-cli profile `my@cgtale.com`)

## 2. The app in one page

| Fact | Value |
|---|---|
| Upstream project | https://github.com/deepseek-ai/deepseek-harness (`master`) |
| Upstream docs | https://deepseek-harness.github.io/deepseek-harness/ |
| Upstream site | https://deepseek.com/harness |
| Upstream status | *developer preview* — rapidly iterating, compatibility-breaking changes expected |
| Community image | `smanx/deepseek-harness:<upstream-version>` (Docker Hub; `ghcr.io/smanx/deepseek-harness` mirror) |
| Image variants | `latest`-style prefixless = slim (Node + DSH + proxy); `devtools-min-*` adds git/curl/jq/pnpm/uv; `devtools-*` adds build toolchain; `admin-*` adds a version-picker service |
| Ports inside the pod | dsh web `127.0.0.1:3079` → community proxy `0.0.0.0:3080` (HTTP + WS, patches loopback check & `crypto.randomUUID` polyfill, handles launch-token handshake) → chart adds `ui-shim` nginx `:3090` |
| State | `$HOME/.dsh` redirected to the appData userspace volume; survives upgrades & reinstalls without `--delete-data` |
| Model providers | Configured in Web UI **Settings → Models** (DeepSeek API key, any OpenAI/Anthropic-compatible endpoint, custom providers with model discovery) |

### Request chain (why the chart looks like this)

```
browser → Olares entrance (Authelia, authLevel private)
        → ui-shim nginx :3090   (strips Accept-Encoding so upstream serves identity-encoded JS;
                                 keeps Settings → Models reachable over the Olares domain)
        → community proxy :3080 (loopback patch + UUID polyfill + optional HTTP Basic Auth)
        → dsh web 127.0.0.1:3079
```

Do **not** remove `ui-shim` unless you have verified the Settings pages work over the Olares
domain without it — that was the #1 "app is broken" symptom historically.

## 3. Version pairs (all must move together)

| Field | File | Rule |
|---|---|---|
| `version` | `dsh/Chart.yaml` | **==** `metadata.version` in OlaresManifest.yaml and the PR title version |
| `metadata.version` | `dsh/OlaresManifest.yaml` | Chart/package version (semver) |
| `appVersion` | `dsh/Chart.yaml` | tracks upstream |
| `spec.versionName` | `dsh/OlaresManifest.yaml` | upstream version string users see |
| image tag | `dsh/templates/deployment.yaml` (`smanx/deepseek-harness:…`) | upstream version actually shipped |

## 4. Upgrade runbook (upstream → Olares)

1. **Check upstream:** repo `releases` for the newest tag; Docker Hub
   `smanx/deepseek-harness` tags for the matching image + arch list
   (`curl -s https://hub.docker.com/v2/repositories/smanx/deepseek-harness/tags?page_size=25`).
2. **Read release notes** for breaking changes that touch:
   ports (`3079/3080`), the home dir (`.dsh`), provider config, web-services behavior.
3. **Edit the four version fields** (§3) + image tag. Only bump `metadata.version`/`Chart.yaml
   version` — a patch bump per shipping change (never bump to retry an arch rejection).
4. `olares-cli chart lint ./dsh` — must print OK.
5. `olares-cli chart package ./dsh -o .` → produces `dsh-<chart-version>.tgz`.
6. Upload + install on the proof instance:
   ```sh
   olares-cli market upload ./dsh-<v>.tgz
   olares-cli market install dsh -s upload --version <v> --watch   # or `upgrade` for existing installs
   ```
7. Verify: `olares-cli market status dsh` → `running`; open the entrance URL
   (`olares-cli settings apps get dsh -o json`), hard-refresh once (`Ctrl+Shift+R`),
   confirm one session works end-to-end and **Settings → Models loads**.
8. Commit + push, publish the GitHub release with the tgz attached.
9. For the public Market: `UPDATE` PR to `beclab/apps` (title `[UPDATE][dsh][<chart-version>] …`),
   version bump mandatory, re-lint before pushing. Keep i18n copies synchronized (§6).

## 5. Design invariants (don't regress these)

- **uid 1000 everywhere** (`spec.runAsUser: true` + pod securityContext; writes go to userspace
  appData only). Never add an explicit root securityContext — OPA/lint reject it; the only root
  part is the `init-permissions` initContainer with the trusted `beclab/` busybox image.
- **`strategy: Recreate`** — appData is a `hostPath` (node-local); rolling updates are incompatible.
- **`options.apiTimeout: 0`** — LLM streams/agent runs exceed the default 15 s entrance cap.
- **Secrets, not literals:** PROXY_USERNAME / PROXY_PASSWORD flow through the
  `dsh-proxy-auth` Secret (`secretKeyRef`); never put credentials into env literals,
  values.yaml commits, or the repo. The chart must carry no ambient credential
  (provider/API keys belong to the user's DSH config in `$HOME/.dsh`.
- **`enableServiceLinks: false`** — avoid legacy service-link env collisions.
- **`/app` pre-warm copy (emptyDir `app-dir`)** — upstream 0.1.6+ entrypoint hardcodes
  `/app/.dsh-web.log` (the bundled proxy's launch-token fisher reads it) and `/app` is
  root-owned in the image while the container must run as uid 1000. The `prep-app`
  init container (same image, uid 1000, no root) copies `/app/.` into the emptyDir and
  the main container shades `/app` with it. Keep `image` in `prep-app` in sync with the
  main container tag on every upgrade. CrashLoopBackOff with
  `cannot create /app/.dsh-web.log: Permission denied` means this wiring broke.
- **`workloadReplicas.dsh: 1`** wired to `{{ .Values.workloads.dsh.replicaCount }}` — required
  for two-phase install, suspend/resume to work.
- **Entrance stays `authLevel: private`** — the harness executes agent plugins; it must remain
  behind Olares entrance auth. "Public app" means listed in the public catalog, not a public URL.
- **`metadata.icon` must be a resolving `http(s)` URL** — currently
  `https://raw.githubusercontent.com/abidals/DeepSeek-Harness-Olares/main/assets/icon/dsh-256.png`.
  Keep the icon + source in `assets/icon/`.

## 6. Security posture & sandbox review

- Single non-privileged workload, uid/gid 1000, namespace `<dsh>-<owner>`, no RBAC/service
  account, no host mounts beyond the userspace `appData` path, capabilities dropped, seccomp
  `RuntimeDefault`, no privilege escalation.
- The bundle executes agent plugins in-process (that is the app's nature). Data isolation relies
  on: per-app namespace, per-user entrance auth (private), per-app appData volume, and the fact
  that the workload can't see other users' mounts. Provider API keys are stored by upstream under
  `$HOME/.dsh` inside appData (app-private volume, backed up, not shared).
- **Local AI API access (reviewed):** dsh is a *model client*. It needs no special platform wiring:
  any user-entered OpenAI-compatible provider works. To reach local models, users add a custom
  provider in **Settings → Models** pointing at an Olares model app's in-cluster service
  (`<svc>.<app>-shared:<port>` for v3 shared model apps, e.g. Ollama/llama.cpp engine bases).
  Egress in-cluster is allowed by default; no NetworkPolicy change is required by this chart.
  Do NOT bake any provider key or endpoint into the chart.
- Future hardening ideas (optional, only if regressions appear):
  `readOnlyRootFilesystem` is intentionally NOT set (npm plugin installs write into the
  container).

## 7. Known-good footguns

| Symptom | Cause | Fix |
|---|---|---|
| Settings → Models unavailable / blank over Olares domain | compression guard in bundled proxy served compressed JS that broke its own patches | keep `ui-shim` (Accept-Encoding strip) |
| WS stuck "connecting" on LAN/remote origin | `crypto.randomUUID` unavailable in non-secure contexts | community proxy injects the polyfill — keep the proxy in the chain |
| Permission denied writing state | uid mismatch: appData owned 1000:1000, process not 1000 | keep `spec.runAsUser: true` + init chown |
| Pod CrashLoop with `exec format error` | image tag lacks the node arch | image is multi-arch; if a variant is used (`devtools-*`), verify that tag's arches |
| CrashLoopBackOff `cannot create /app/.dsh-web.log: Permission denied` | entrypoint writes into root-owned `/app`; uid 1000 cannot | keep the `prep-app` emptyDir pre-warm wiring (see §5) |
| Install stuck at download | registry unreachable / bad image tag | `market status` then doctor; never "retry" with only a version bump if bytes didn't change |
| Long requests cut at 504 | entrance proxy timeout | keep `apiTimeout: 0` |
| 422 `appenv` at install | env validation | check `envs[]` contract — both PROXY_* are optional (`required: false`) |

## 8. Publishing & distribution

1. **This repo (abidals/DeepSeek-Harness-Olares)** — source of truth for Olares users + asset hosting:
   commit the chart + assets, tag `v<chart-version>`, attach `dsh-<chart-version>.tgz` to the release.
   Icon/promo images are served from `raw.githubusercontent.com` — URLs baked into the manifest.
2. **Public Market (beclab/apps)** — `ADD`/`UPDATE` PR from a fork of https://github.com/beclab/apps:
   - the `dsh/` folder (not the tgz) + `owners` file go at the fork root, folder name stays `dsh`
   - PR title `[NEW][dsh][<chart-version>] Add DeepSeek Harness` (or `UPDATE`)
   - category values must be Market-valid (`AI`); no `.suspend`/`.remove` files in the OAC root
   - start as Draft, click Ready for review once GitBot is quiet
   - after merge the app indexes into `market.olares` shortly; each user installs their own
     private (`authLevel: private`) instance, synced per device via their own LarePass login
   - NOTE (2026-09-28): the current `abidals` fine-grained PAT is repo-scoped and **cannot
     fork or create repositories**, so the first fork step is manual: fork beclab/apps in the
     browser (as abidals), add a branch, copy this repo's `dsh/` folder in, push, then open the
     draft PR to `beclab/apps:main`. To automate next time, issue a PAT with repository
     administration (create/fork) permission.
3. **Uninstall data safety:** userspace Data volume is only deleted with
   `market uninstall --delete-data`; plain uninstall keeps `$HOME/.dsh` state.

## 9. Open threads / next ideas

- Add real in-app screenshots to `assets/listing/` and swap into the manifest via `UPDATE` PR
  (current images are deterministic brand composites; no fake UI).
- Consider an automated "image tag watcher" — check Docker Hub tags and open an issue when
  upstream ships a new release.
- If upstream ever fixes the remote-origin Settings behavior, drop the `ui-shim` sidecar
  (verify the fix from the Olares domain first!).
- Watch for the `admin-*` image variant if a built-in version picker is ever wanted.