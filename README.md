# Nemotron 3 Nano on vLLM: 2-GPU Production-Readiness Benchmark

This repository publishes a sanitized production-readiness benchmark for
`nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B-BF16` served by vLLM on a 2-GPU
deployment.

The goal is not to rank model quality. The goal is to answer a serving
question:

> Under a short-context, long-output streaming workload, where is this
> deployment interactive, where is it internally usable, and where does
> it become batch-only?

Internal service endpoints, Kubernetes namespaces, private hostnames,
local filesystem paths, credentials, bearer tokens, and generated model
response bodies were removed from this public package.

## TL;DR

Controlled GO for the benchmarked workload.

- Interactive chat is supported up to around `2 rps` offered load, with
  `~1.80 completed requests/s`, p95 TTFT `533 ms`, p95 e2e `6.1s`, and
  `0%` errors.
- High-throughput internal usage remains practical around `5 rps`
  offered load, with p95 TTFT `646 ms`, p95 e2e `12.1s`, and
  `~1992 client-observed output tokens/s`.
- The main queueing boundary appears between `5` and `6 rps`: observed
  inflight requests move from `61` to `93`, while server-side
  `maxRequestsRunning` reaches the configured `max_num_seqs=64` region
  and `maxRequestsWaiting` rises from `0` to `19`.
- The deployment remains stable at `10 rps` offered load with `0%`
  errors and `0` timeouts, but p95 TTFT reaches `17.3s`, making this
  batch capacity rather than interactive capacity.
- The bottleneck at high offered load is queueing before first token,
  not per-token decode speed: p95 ITL stays around `24-25 ms` at
  `6-10 rps` while TTFT increases sharply.

## Decision

**Controlled GO** for this specific 2-GPU vLLM deployment under the
benchmarked short-context, decode-heavy streaming workload.

Do use this result for:

- sizing interactive traffic near the `2 rps` offered-load region;
- routing tolerant internal assistant traffic near the `5 rps`
  offered-load region;
- treating `8-10 rps` offered load as batch / agent-job capacity when
  multi-second TTFT is acceptable.

Do not use this result to claim:

- `10 rps` interactive serving;
- long-context interactive readiness;
- model-quality superiority;
- multi-tenant autoscaling behavior;
- cost parity with hosted APIs;
- generalization to other GPUs, vLLM versions, model revisions, or
  serving parameters.

## Operating Envelope

| Workload class | Recommended operating point | Client evidence | Server evidence | Production reading |
| --- | ---: | --- | --- | --- |
| Interactive chat | `<= 2 rps` offered load, `~1.80 completed rps` observed | p95 TTFT `533 ms`, p95 e2e `6.1s`, `100%` TTFT < `1s` | max running `11`, max waiting `0` | GO |
| High-throughput internal assistant | around `5 rps` offered load, `~3.90 completed rps` observed | p95 TTFT `646 ms`, p95 e2e `12.1s`, `~1992 output tokens/s` | max running `58`, max waiting `0` | GO with monitoring |
| Queueing boundary | between `5` and `6 rps` offered load | p95 TTFT jumps from `646 ms` to `5.47s`; p95 e2e rises to `16.1s` | max running hits `64`; max waiting rises to `19` | admission control recommended |
| Batch / agent jobs | `8-10 rps` offered load | `0%` errors, p95 TTFT `12.2-17.3s`, output TPS `~2203-2205` | max running `64`; max waiting `41-58`; throttling active `0` | batch only |

## Main Finding

This deployment is not a `10 rps` interactive chat service. It is a
stable 2-GPU vLLM serving configuration with a clear operating envelope:

1. Up to `2 rps` offered load: interactive latency remains strong.
2. Around `5 rps` offered load: throughput is high and TTFT remains
   below `1s` p95.
3. Between `5` and `6 rps`: queueing becomes the dominant factor.
4. At `8-10 rps`: the system remains stable, but the workload is
   batch-only because users wait many seconds before the first token.

The new server-side telemetry confirms the interpretation. At `5 rps`,
`maxRequestsRunning` is `58` and `maxRequestsWaiting` is `0`. At `6 rps`,
`maxRequestsRunning` reaches `64` and `maxRequestsWaiting` becomes `19`.
At `10 rps`, `maxRequestsRunning` remains capped at `64` while
`maxRequestsWaiting` reaches `58`.

That is the operational wall: once request pressure crosses the serving
capacity region, vLLM continues serving without errors, but additional
requests wait before first token.

## Key Evidence

Source: [`data/request-rate-combined-summary.csv`](data/request-rate-combined-summary.csv)

| Offered RPS | Completed RPS | Max inflight | p95 TTFT | p95 e2e | Client output TPS | Error rate | Reading |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: | --- |
| 2 | 1.7979 | 13 | 533 ms | 6111 ms | 880.7 | 0% | interactive |
| 5 | 3.9044 | 61 | 646 ms | 12119 ms | 1992.0 | 0% | high-throughput internal |
| 6 | 3.7836 | 93 | 5466 ms | 16121 ms | 1930.9 | 0% | queueing begins |
| 8 | 4.3238 | 133 | 12219 ms | 21588 ms | 2203.5 | 0% | batch leaning |
| 10 | 4.3194 | 169 | 17306 ms | 26761 ms | 2205.3 | 0% | batch only |

Important: the benchmark generated up to `10 rps` of offered load. Under
this load, the deployment completed approximately `4.32 requests/s` with
`0%` errors and `0` timeouts, but p95 TTFT reached `17.3s`.

## Server-Side Evidence

Source: [`data/server-metrics-deltas.csv`](data/server-metrics-deltas.csv)

Server metrics were collected through an authenticated Prometheus API
scrape during the benchmark. The scrape included vLLM scheduler metrics,
vLLM latency/token counters, DCGM GPU telemetry, and nvidia-smi
throttling metrics.

| Offered RPS | Max running | Max waiting | Avg server TTFT | Avg server e2e | Avg server ITL | Max GPU util | Max power | VRAM used | Throttle active |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 2 | 11 | 0 | 0.060s | 5.49s | 10.6 ms | 100% | 711 W | 178242 MiB | 0 |
| 5 | 58 | 0 | 0.095s | 10.73s | 20.8 ms | 100% | 688 W | 178242 MiB | 0 |
| 6 | 64 | 19 | 1.948s | 13.49s | 22.6 ms | 97.5% | 628 W | 178242 MiB | 0 |
| 8 | 64 | 41 | 5.174s | 16.41s | 22.0 ms | 100% | 681 W | 178242 MiB | 0 |
| 10 | 64 | 58 | 7.606s | 18.95s | 22.2 ms | 100% | 700 W | 178242 MiB | 0 |

The strongest diagnostic is the divergence between queue-related metrics
and per-token latency:

- `maxRequestsRunning` caps at `64`, matching the serving configuration
  region.
- `maxRequestsWaiting` rises from `0` at `5 rps` to `58` at `10 rps`.
- Average server ITL remains around `22 ms` at high offered load.
- nvidia-smi throttling stays inactive in the sampled telemetry.

This is queueing before first token, not a decode-speed collapse.

## Goodput / SLO Pass Rate

Source: [`data/goodput-slo-summary.csv`](data/goodput-slo-summary.csv)

The thresholds below are illustrative production SLOs, not universal
latency requirements.

| Offered RPS | Completed RPS | Error-free | TTFT < 1s | TTFT < 5s | TPOT < 25ms | e2e < 15s | Reading |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: | --- |
| 2 | 1.7979 | 100% | 100% | 100% | 100% | 100% | interactive |
| 5 | 3.9044 | 100% | 100% | 100% | 100% | 100% | high-throughput internal |
| 6 | 3.7836 | 100% | 35.42% | 88.75% | 92.08% | 64.58% | queueing begins |
| 8 | 4.3238 | 100% | 26.67% | 53.33% | 100% | 30.42% | batch leaning |
| 10 | 4.3194 | 100% | 26.67% | 31.25% | 100% | 26.67% | batch only |

The most important diagnostic is that TTFT and e2e SLOs collapse before
TPOT does. The system can still decode tokens steadily once a request is
admitted, but many requests wait too long before admission.

## Methodology

| Field | Value |
| --- | --- |
| Run date | 2026-06-03 |
| Benchmark mode | open-loop request-rate sweep |
| Endpoint type | OpenAI-compatible streaming endpoint |
| Workload | short prompt, long generation, decode-heavy |
| Model | `nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B-BF16` |
| Model snapshot | `cbd3fa9f933d55ef16a84236559f4ee2a0526848` |
| Max output tokens | `512` |
| Average generated output length | approximately `512` output tokens per completed request across high-rate levels |
| Request-rate levels | `0.25, 0.5, 0.75, 1, 1.25, 1.5, 2, 3, 4, 5, 6, 8, 10 rps` |
| Measured requests | `2310` |
| Warmup | `3` warmup requests, excluded from measured totals |
| Duration per level | `120000 ms` or request-limit completion, whichever came first |
| Cooldown between levels | `60000 ms` |
| Sampling | `temperature=0`, `top_p=1` |
| Reasoning mode | `enable_thinking=false` |
| Streaming | `true` |
| Timeout | `120000 ms` |
| Server telemetry | authenticated Prometheus API scrape every `2000 ms` |
| Server runtime | vLLM `0.21.0`, PyTorch `2.11.0+cu130`, NVIDIA driver `595.58.03`, CUDA runtime `13.2` |
| Server hardware | 2 x NVIDIA RTX PRO 6000 Blackwell Server Edition, tensor parallel size `2` |

The benchmark was executed through the same private serving path used by
internal clients. It therefore measures the operational path, not a
localhost-only microbenchmark.

## Metric Definitions

| Metric | Definition |
| --- | --- |
| Offered RPS / target RPS | Rate at which the open-loop client initiates new requests. It is not the same as completed throughput under saturation. |
| Completed RPS / actual RPS | Completed measured requests divided by elapsed benchmark time for the level. |
| TTFT | Time from request start until the first streamed token is received by the client. |
| e2e latency | Time from request start until the final token / completion is received by the client. |
| ITL / TPOT | Inter-token latency, measured as average delay between successive output tokens during streaming. |
| Client output TPS | Client-side output token estimate divided by benchmark duration. This is the primary throughput reading in the request-rate table. |
| Server delta output TPS | vLLM generation-token counter delta divided by the Prometheus snapshot interval. This is useful for corroboration, but its denominator differs from the client benchmark duration. |
| `num_requests_running` | vLLM scheduler count of requests actively being served. |
| `num_requests_waiting` | vLLM scheduler count of requests waiting for service. |
| Error rate | HTTP/client errors plus timeouts divided by measured requests. |

## Server And vLLM Profile

The public manifest keeps only benchmark-relevant hardware and runtime
metadata. It removes private hostnames, paths, services, credentials, and
deployment details.

| Domain | Public benchmark value |
| --- | --- |
| Runtime | vLLM `0.21.0`, PyTorch `2.11.0+cu130`, NVIDIA driver `595.58.03`, CUDA runtime `13.2` |
| GPUs | 2 x NVIDIA RTX PRO 6000 Blackwell Server Edition, 97887 MiB VRAM per GPU |
| Parallelism | tensor parallel `2`, pipeline `1`, data parallel `1` |
| vLLM limits | `max_model_len=32768`, `max_num_seqs=64`, `gpu_memory_utilization=0.90` |
| Scheduler | async scheduling and chunked prefill enabled |
| CPU/RAM | 32 logical CPUs, 251 GiB RAM |

`max_num_seqs=64` is the key interpretation threshold. TTFT stays below
1 second while observed pressure remains near or below this serving
capacity region. Once pressure exceeds it, vLLM keeps serving without
errors but introduces queueing before first token.

## Operational Interpretation

Recommended routing policy for this workload:

- Keep interactive chat near or below the `2 rps` offered-load region.
- Route tolerant internal assistant workloads around the `5 rps`
  offered-load region, with monitoring on `num_requests_waiting` and
  p95 TTFT.
- Treat `8-10 rps` offered load as batch / agent-job capacity only.
- Use admission control or a rate limiter to protect interactive traffic
  once `num_requests_running` approaches `64` or
  `num_requests_waiting > 0`.
- Do not mix long-context interactive requests into the same pool without
  a separate benchmark; this run is short-context and decode-heavy.

## Known Limitations

This benchmark provides production-readiness evidence for one workload
and one deployment. It is not a universal serving certificate.

Current limitations:

- no 30-60 minute soak test;
- no mixed workload test combining chat, long-context, JSON, and batch;
- no multi-tenant autoscaling test;
- no long-context interactive latency benchmark;
- no model-quality comparison against other models;
- no cost/API parity analysis;
- no energy-per-token or watts-per-million-token calculation;
- no public raw generated response bodies.

Near-32k long-context rejections in broader internal testing were server
budget rejections, not model failures: `input_tokens + output_tokens`
exceeded the configured `max_model_len=32768`.

## Next Benchmark Roadmap

The next run should broaden production coverage:

- run a 30-60 minute soak at the recommended interactive and batch
  ceilings;
- add long-context interactive cases at `8k`, `16k`, and `30k` input
  tokens;
- add a mixed traffic benchmark with separate chat, batch, JSON, and
  agent workloads;
- publish p99-focused operating envelopes;
- add energy and cost estimates if the underlying data can be shared;
- compare routing policies that isolate interactive and batch traffic.

## Data Files

| Path | Content |
| --- | --- |
| [`data/request-rate-combined-summary.csv`](data/request-rate-combined-summary.csv) | Request-rate summary from `0.25` to `10 rps` for the 2026-06-03 authenticated-metrics run |
| [`data/goodput-slo-summary.csv`](data/goodput-slo-summary.csv) | Per-level SLO pass rates computed from measured request records |
| [`data/server-metrics-deltas.csv`](data/server-metrics-deltas.csv) | Per-level vLLM/GPU metric deltas from authenticated Prometheus scrapes |
| [`data/reference-runs/`](data/reference-runs/) | Sanitized metrics-only appendices for earlier smoke, quality, long-context, stress, and closed-loop throughput runs |
| [`DATA_NOTICE.md`](DATA_NOTICE.md) | Sanitization and inclusion notice |

The `reference-runs` directory is included for traceability. It is not
the primary request-rate score. Each run directory contains a public
manifest, aggregate metrics, stress profile when applicable, and JSONL
metrics-only records. Raw generated response bodies are not included.

## Efficiency Metrics

The public artifacts support these simple efficiency readings based on
client-observed output throughput:

| Offered load | Client output TPS | GPUs | Output TPS/GPU | Completed RPS/GPU |
| ---: | ---: | ---: | ---: | ---: |
| `2 rps` | `880.7` | 2 | `440.3` | `0.899` |
| `5 rps` | `1992.0` | 2 | `996.0` | `1.952` |
| `10 rps` | `2205.3` | 2 | `1102.6` | `2.160` |

Cost per million tokens and watts per million tokens are intentionally
not reported because the public export does not include amortized
hardware cost or energy integration.

## Data Reuse And Citation

This dataset is released under the terms described in [`LICENSE`](LICENSE).
Citation metadata is provided in [`CITATION.cff`](CITATION.cff).

## Public-Safety Scope

This export intentionally excludes:

- API keys, bearer tokens, Kubernetes secrets, and credentials;
- internal service URLs, cluster namespaces, private hostnames, and local
  filesystem paths;
- Kubernetes manifests, deployment files, and codebase source files;
- generated model response bodies from raw stress JSONL files.

The package is meant to be published as benchmark evidence, not as an
operational runbook for the private infrastructure used to run it.
