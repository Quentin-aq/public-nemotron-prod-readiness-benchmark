# Public Data Notice

This folder is a sanitized public export of the 2026-06-02 Nemotron 3 Nano production-readiness benchmark.

Included:
- Aggregate CSV summaries.
- Goodput / SLO pass-rate CSV computed from per-request measured records.
- Public manifests reconstructed from the original manifests.
- Stress profiles and aggregate metrics.
- Per-request JSONL metrics with generated response text removed.
- Server metric snapshots with internal endpoints removed.
- Reference-run appendices for smoke, quality, long-context, stress, and closed-loop throughput runs, exported as public manifests and metrics-only data.
- SVG charts referenced by the README.

Excluded or redacted:
- API keys, Kubernetes secrets, bearer tokens, credentials.
- Internal service URLs, cluster namespaces, private hostnames, local filesystem paths.
- Generated model response bodies from raw stress JSONL files.
- Kubernetes job manifests and codebase files.
