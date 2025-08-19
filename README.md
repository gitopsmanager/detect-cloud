# Detect Cloud Provider (self-hosted)

> **Status: Beta** — use `@main` until the first stable release.

Detect whether a self-hosted GitHub Actions runner is running on **Azure** or **AWS** — fast, zero-dependency, and safe.  
The action probes cloud **IMDS** (instance metadata) endpoints first and falls back to lightweight environment heuristics.

Returns one of: `azure`, `aws`, or `unknown`.

---

## Features

- **Zero deps:** Bash + `curl` only (no Python, no kubectl).
- **Fast path:** Azure IMDS checked first (instant on AKS), then a single AWS IMDSv2 token probe.
- **Network-safe:** Only contacts `169.254.169.254`; proxies bypassed via `NO_PROXY`.
- **Deterministic:** Exposes both the detected `provider` and the `method` used.

---

## How it works

1. **Azure IMDS probe**  
   `GET http://169.254.169.254/metadata/instance/compute?api-version=2021-02-01` with header `Metadata: true`.  
   If 200 ⇒ `provider=azure`.

2. **AWS IMDSv2 probe** (single request)  
   `PUT http://169.254.169.254/latest/api/token` with `X-aws-ec2-metadata-token-ttl-seconds`.  
   If 200 ⇒ `provider=aws`.

3. **Heuristics (last resort)**  
   If IMDS isn’t reachable, use env hints:
   - `AWS_REGION` / `AWS_DEFAULT_REGION` ⇒ `aws`
   - `MSI_ENDPOINT` / `IDENTITY_ENDPOINT` ⇒ `azure`
   - Otherwise ⇒ `unknown`

Each probe uses a configurable timeout (default **1000 ms**) and returns quickly when endpoints respond with non-matching status codes.

---

## Inputs

| Name         | Type   | Default | Description                                                                 |
|--------------|--------|---------|-----------------------------------------------------------------------------|
| `timeout-ms` | string | `"1000"`| Per-probe timeout (milliseconds). Rounded up to whole seconds for `curl`.  |

---

## Outputs

| Name       | Values                                        | Description                                             |
|------------|-----------------------------------------------|---------------------------------------------------------|
| `provider` | `azure` \| `aws` \| `unknown`                 | The detected platform.                                  |
| `method`   | `imds:azure` \| `imds:aws-token` \| `env:*`   | How detection succeeded (useful for logs/diagnostics).  |

---

## Usage

> Place `action.yml` at the **root** of this repository for Marketplace-friendly usage.

```yaml
jobs:
  detect:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v4

      - name: Detect cloud
        id: cloud
        uses: affinity7software/detect-cloud@main   # Beta channel
        with:
          timeout-ms: 800

      - name: Print result
        run: echo "Provider=${{ steps.cloud.outputs.provider }} via ${{ steps.cloud.outputs.method }}"

      - name: Azure setup
        if: steps.cloud.outputs.provider == 'azure'
        run: echo "Do Azure things"

      - name: AWS setup
        if: steps.cloud.outputs.provider == 'aws'
        run: echo "Do AWS things"

```

## Timing & timeouts

On Azure, the AWS probe typically fails immediately (non-200), so it won’t sit on the full timeout.

On AWS, the Azure probe fails quickly; the AWS token probe returns 200.

Worst case (IMDS blackholed by network policy): a probe may take up to timeout-ms.  
Tip: set timeout-ms: 500 for hardened networks.

---

## Requirements

Linux runner with Bash + curl (standard on most self-hosted runners, including ARC).

No credentials required. Read-only metadata checks only.

Proxies: The action sets NO_PROXY/no_proxy to avoid proxying IMDS calls.

---

## Troubleshooting

**provider=unknown**

IMDS blocked or unreachable from container/network.

Env heuristics not present.  
Options: allow IMDS, lower timeout-ms, or inject explicit envs (e.g., AWS_REGION or MSI_ENDPOINT) if acceptable.

**Slow detection**

Lower timeout-ms (e.g., 500).

Ensure 169.254.169.254 is either reachable or cleanly rejected (no blackhole).

---

## Versioning

Beta: Pin to @main while testing.

After stabilization, releases will follow SemVer (e.g., v1.0.0) with a floating major tag v1.

---

## Contributing

PRs welcome! Please:

- Keep the script dependency-free (Bash + curl).
- Include a short rationale for detection/timing changes.
- Add/update simple CI checks where feasible.

---

## License

MIT © 2025 Affinity7 Consulting Ltd  
See LICENSE for details.

