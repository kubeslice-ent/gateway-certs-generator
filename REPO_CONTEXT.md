# REPO_CONTEXT: gateway-certs-generator

## Purpose
A single-purpose Kubernetes Job image that generates OpenVPN TLS certificates and keys for KubeSlice slice gateways, then stores the resulting secrets directly into the cluster via the Kubernetes API. It combines shell-based `easy-rsa` PKI management with a Go binary that writes the final artifacts as Kubernetes Secrets.

## Role in EGS System
The `kubeslice-controller` spawns this image as a Kubernetes Job (configured under `ovpnJob` in the controller Helm values) whenever a new SliceGateway pair needs to be provisioned. The job produces two Kubernetes Secrets per gateway pair — one for the OpenVPN server and one for the client — which the `worker-operator` mounts into the gateway pods to establish encrypted inter-cluster tunnels for KubeSlice network slices.

## Tech Stack
- **Language:** Go 1.17 (binary `generator`), POSIX shell (cert generation scripts)
- **Framework:** None (plain Go binary + shell scripts)
- **Key dependencies:**
  - `k8s.io/client-go v0.22.1` — in-cluster Kubernetes API access to create Secrets
  - `k8s.io/api v0.22.1`, `k8s.io/apimachinery v0.22.1` — Kubernetes API types
  - `go.uber.org/zap v1.21.0` — structured logging
  - `easy-rsa v3.0.8` (bundled) — PKI/CA and certificate management
  - `openvpn`, `openssl`, `jq` (Alpine packages) — VPN key generation and JSON parsing

## Key Components
```
gateway-certs-generator/
├── main.go                         # Go binary: reads generated files, writes K8s Secrets
├── util/
│   └── common.go                   # Zap log-level helper
├── generate-certs.sh               # Entry-point shell script (container CMD)
├── ovpn/
│   ├── easyrsa-v3.0.8/             # Bundled EasyRSA v3.0.8 CLI + openssl config + x509 types
│   ├── scripts/
│   │   ├── initialize.sh           # Variable init (PASS_SALT, PASSIN, PASSOUT)
│   │   ├── setEnv.sh               # Path constants (CA_CERT, TA_KEY)
│   │   ├── validate.sh             # Input validation + parameter printing
│   │   └── process.sh              # Core cert generation logic
│   └── template/
│       ├── server-openvpn.conf     # Server OpenVPN config template
│       ├── client-openvpn-combined.conf  # Single-file combined client .ovpn template
│       ├── ovpn_env.sh             # Environment variable declarations for OpenVPN
│       └── ccd                     # Client config directory template (iroute + ifconfig-push)
├── bootstrap.sh                    # Dev bootstrap (apt-get openvpn jq)
├── Dockerfile                      # Two-stage build: golang:1.17-alpine3.16 → alpine:3.16
├── Makefile                        # docker-build, chart-deploy, chart-undeploy targets
├── go.mod / go.sum                 # Go module: github.com/kubeslice/gateway-certs-generator, Go 1.17
├── Jenkinsfile                     # Jenkins pipeline via shared library (opensource-release)
└── .github/workflows/
    ├── docker-build.yaml           # GitHub Actions: manual docker build (workflow_dispatch)
    └── trivy.yml                   # GitHub Actions: Trivy vulnerability scan
```

## Execution Flow

The container runs `generate-certs.sh` as its CMD:

1. **`init`** — Sets internal PKI pass-phrase salts (`PASS_SALT`, `PASSIN`, `PASSOUT`).
2. **`validateInput`** — Checks all required env vars: `NAMESPACE`, `SERVER_SLICEGATEWAY_NAME`, `CLIENT_SLICEGATEWAY_NAME`, `SLICE_NAME`, `CERT_GEN_REQUESTS` (base64-encoded JSON).
3. **`prepareExecution`** — Copies `ovpn/` tree to `$WORK_DIR`, sets EasyRSA env vars.
4. **`initPkiDirectory`** — Runs `easyrsa init-pki`.
5. **`generateDhPemFileIfMissing`** — Generates `dh.pem` via `openssl dhparam 2048`.
6. **`generateTaKeyFileIfMissing`** — Generates `ta.key` via `openvpn --genkey --secret`.
7. **`generateCaCertsIfMissing`** — Runs `easyrsa build-ca` with CN `AveshaCA`.
8. **`iterateCertPairRequest processIndividualCertRequest`** — For each cert pair: generates server + client certs, substitutes template parameters (FQDN, network CIDRs, VPN IPs, protocol, cipher), assembles final artifact directory.
9. **`./generator`** (Go binary) — Reads generated files from disk and writes two Kubernetes Secrets via in-cluster API.

## Kubernetes Secrets Written

| Secret | Key | Content |
|---|---|---|
| `CLIENT_SLICEGATEWAY_NAME` | `ovpnConfigFile` | Combined client `.ovpn` config |
| `SERVER_SLICEGATEWAY_NAME` | `ovpnConfigFile` | Server `openvpn.conf` |
| `SERVER_SLICEGATEWAY_NAME` | `pkiDhPemFile` | DH params (`dh.pem`) |
| `SERVER_SLICEGATEWAY_NAME` | `pkiTAKeyFile` | TLS-auth key (`ta.key`) |
| `SERVER_SLICEGATEWAY_NAME` | `pkiIssuedCertFile` | Server certificate (`.crt`) |
| `SERVER_SLICEGATEWAY_NAME` | `pkiPrivateKeyFile` | Server private key (`.key`) |
| `SERVER_SLICEGATEWAY_NAME` | `pkiCACertFile` | CA certificate (`ca.crt`) |
| `SERVER_SLICEGATEWAY_NAME` | `ccdFile` | Client config directory file |

## Key Environment Variables

| Variable | Description |
|---|---|
| `NAMESPACE` | Kubernetes namespace for Secrets |
| `SERVER_SLICEGATEWAY_NAME` | Name for the server-side Kubernetes Secret |
| `CLIENT_SLICEGATEWAY_NAME` | Name for the client-side Kubernetes Secret |
| `CERT_GEN_REQUESTS` | Base64-encoded JSON array of cert-pair descriptors (`vpnFqdn`, `serverId`, `clientId`, network CIDRs, VPN IPs) |
| `WORK_DIR` | Working directory for PKI artifacts (default: `/work`) |
| `SRC_DIR` | Source directory with bundled templates (default: `/app`) |
| `LOG_LEVEL` | Go binary log verbosity (`debug`, `info`, `error`) |

## Dependencies & Integrations
- **kubeslice-controller** (caller): creates this Job when provisioning a new SliceGateway pair; sets all env vars; Job ServiceAccount must have `create`/`delete` Secret permissions in the slice gateway namespace
- **worker-operator** (consumer): mounts the generated Secrets into OpenVPN gateway pods on worker clusters to establish encrypted inter-cluster tunnels
- **Docker Hub** (`aveshasystems/gateway-certs-generator`): published container image referenced in the controller Helm values under `kubeslice.ovpnJob.image`
