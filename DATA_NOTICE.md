# Public Data Notice

This folder is a sanitized public export of the 2026-06-02 Nemotron 3
Nano production-readiness benchmark.

Included:

- aggregate CSV summaries;
- Goodput / SLO pass-rate CSV computed from per-request measured records;
- public manifests reconstructed from the original manifests;
- stress profiles and aggregate metrics;
- per-request JSONL metrics with generated response text removed;
- server metric snapshots with internal endpoints removed;
- reference-run appendices for smoke, quality, long-context, stress, and
  closed-loop throughput runs, exported as public manifests and
  metrics-only data;
- SVG charts referenced by the README.

Excluded or redacted:

- API keys, Kubernetes secrets, bearer tokens, credentials;
- internal service URLs, cluster namespaces, private hostnames, and local
  filesystem paths;
- generated model response bodies from raw stress JSONL files;
- Kubernetes job manifests and codebase files.
