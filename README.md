# kubernetes-base

The generic chart library: reusable, application-agnostic Helm charts for the
Kubernetes fleet. Application composition lives in the language repos
([kubernetes-php](https://github.com/99linesofcode/kubernetes-php)); this repo
holds the building blocks any stack composes.

## Charts

- **`charts/certificates/`** — cert-manager `ClusterIssuer`s: a self-signed
  issuer, a local CA, and a Cloudflare DNS-01 ACME issuer, each gated by a
  values toggle. Install it once per cluster; applications reference
  `local-ca-issuer` (local) or `cloudflare-dns01-issuer` (production) from their
  `Certificate`.

Planned: a `gateway` chart (Traefik in Gateway API mode) and other shared
infrastructure charts.

## Conventions

- Every chart is generic — no application vocabulary in a chart name or a value.
- Values toggle environments; templates never branch on environment.
- Charts are published to `oci://ghcr.io/99linesofcode/charts` and consumed by
  Flux from [kubernetes-fleet](https://github.com/99linesofcode/kubernetes-fleet).

## Usage

```bash
helm lint charts/certificates
helm template charts/certificates
helm install certificates charts/certificates
```

## Contributing

Please review the [Contribution Guidelines](https://github.com/99linesofcode/.github/blob/main/.github/CONTRIBUTING.md).

## Code of conduct

In order to ensure that the community is welcoming to all, please review and abide by the [Code of Conduct](https://github.com/99linesofcode/.github?tab=coc-ov-file).

## License

This software is open source and licensed under the [MIT license](https://github.com/99linesofcode/kubernetes-base?tab=MIT-1-ov-file).
