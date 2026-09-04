# DeepSeek Harness — Olares App

Unofficial [Olares](https://olares.com) app package for [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) (`dsh`), the open-source agent harness by [DeepSeek AI](https://deepseek.com) — *everything is a plugin*. The Web UI gives you sessions, workspaces and model routing in the browser.

Upstream app: [`deepseek-ai/deepseek-harness`](https://github.com/deepseek-ai/deepseek-harness) (MIT, developer preview) — community container image `smanx/deepseek-harness` (multi-arch), version **0.1.2-rc.1**.

## What you get

- Single-container web app served behind your Olares entrance at `https://<app>.<your-domain>`
- All agent state (credentials, settings, sessions) persisted on the Olares userspace Data volume → survives upgrades
- The bundled community proxy handles the upstream launch-token handshake automatically; no browser dance needed
- Optional **HTTP Basic Auth** in front of the Web UI: set `PROXY_USERNAME` / `PROXY_PASSWORD` in Settings → Applications → Manage environment variables (Olares entrance auth stays on top of it)
- A small **compatibility sidecar** (`nginx`): it normalizes response compression so the bundled proxy's browser patches apply over the Olares domain — this is what keeps **Settings → Models** fully usable (without it, the upstream client treats any non-`localhost` origin as a remote browser and disables its settings surface)

## Prerequisites

- An Olares machine running **Olares 1.12.6+**, logged in with your Olares ID
- [`@olares/cli`](https://www.npmjs.com/package/@olares/cli) installed and logged in:
  ```sh
  npm install -g @olares/cli@latest
  olares-cli profile login --olares-id you@example.com
  ```
- A [DeepSeek API key](https://platform.deepseek.com/) (or any OpenAI-compatible endpoint for custom providers)

## Install

1. **Get the chart package** — download `dsh-<version>.tgz` from this repo's [Releases](https://github.com/abidals/DeepSeek-Harness-Olares/releases/latest) (or clone and build it yourself):
   ```sh
   olares-cli market upload ./dsh-0.1.3.tgz
   # building from source instead:
   git clone https://github.com/abidals/DeepSeek-Harness-Olares && cd DeepSeek-Harness-Olares
   olares-cli chart package ./dsh -o .
   ```

2. **Install**:
   ```sh
   olares-cli market install dsh -s upload --version 0.1.3 --watch
   ```

3. **Open the app** from your Olares desktop, then hard-refresh once (`Ctrl+Shift+R`) so the browser fetches the freshly served JS bundle.

4. **Configure a model** — open **Settings → Models**, enter your DeepSeek API key and save; the model route becomes usable immediately without a restart. **Add provider** covers built-ins (`anthropic`, `openai`, `moonshotai`, ...), and **Add a custom provider** takes any OpenAI-compatible gateway (`openai-completions`, `openai-responses` or `anthropic-messages`) with model discovery.

## Updating the app

When upstream ships a new version: bump the image tag in `dsh/templates/deployment.yaml` + `appVersion` in `Chart.yaml`, bump `version` in both `Chart.yaml` and `OlaresManifest.yaml`, then:

```sh
olares-cli chart lint ./dsh
olares-cli chart package ./dsh -o .
olares-cli market upload ./dsh-<new-version>.tgz
olares-cli market upgrade dsh -s upload --version <new-version> --watch
```

Then publish the new package as a GitHub release so others get it too:

```sh
git add -A && git commit -m "bump to <upstream version>" && git push
gh release create v<new-version> ./dsh-<new-version>.tgz --latest
```

Your keys, settings and sessions survive upgrades (the env values and the app-data volume persist).

## Repo layout

```
dsh/                 # the Olares Helm-style chart (OlaresManifest.yaml + templates/)
dsh-0.1.3.tgz        # pre-built chart package (what `market upload` consumes)
```

## Notes

- Request chain: browser → Olares entrance (Authelia) → `nginx` shim `:3090` (strips `Accept-Encoding`) → bundled proxy `:3080` (patches the loopback check + `crypto.randomUUID` polyfill, launch-token handshake) → `dsh web` `:3079`.
- The upstream project is a *developer preview* and iterates rapidly — expect compatibility-breaking changes; this chart pins the image tag per release.
- The bundled proxy listens on `0.0.0.0` inside the pod without TLS — it is only reachable through the cluster service; never expose its port manually.
- Long agent streams and big uploads are safe: the chart sets `options.apiTimeout: 0` so the entrance never cuts in-flight requests.
- Data lives in the per-app userspace volume; uninstalling the app removes the workload — back up `$DSH_HOME` (`.credentials.yaml`, `settings.yaml`, sessions) from the app data dir before uninstalling if you need it elsewhere.

## License

The DeepSeek Harness application is MIT (upstream). This packaging repo carries no separate license claim on the app itself.