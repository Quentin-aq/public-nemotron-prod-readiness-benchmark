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
local filesystem paths, credentials, and generated response bodies were
removed from this public package.

## TL;DR

Controlled GO for the benchmarked workload.

- Interactive chat is supported up to around `2 rps` offered load, with
  `~1.94 completed requests/s`, p95 TTFT around `541 ms`, and `0%`
  errors.
- High-throughput internal usage remains practical around `5 rps`
  offered load, with p95 TTFT still below `1s`.
- The main queueing boundary appears between `5` and `6 rps`, when max
  inflight requests move beyond the configured `max_num_seqs=64` region.
- The deployment remains stable at `10 rps` offered load with `0%`
  errors, but p95 TTFT reaches `16.5s`, making this batch capacity rather
  than interactive capacity.
- The bottleneck at high offered load is queueing before first token, not
  per-token decode speed.

## Decision

**Controlled GO** for this specific 2-GPU vLLM deployment under the
benchmarked short-context, decode-heavy workload.

Do use this result for:

- sizing interactive traffic near the `2 rps` offered-load region;
- routing tolerant internal assistant traffic near the `5-6 rps`
  offered-load region;
- treating `8-10 rps` offered load as batch / agent-job capacity.

Do not use this result to claim:

- `10 rps` interactive serving;
- long-context interactive readiness;
- model-quality superiority;
- multi-tenant autoscaling behavior;
- cost parity with hosted APIs;
- generalization to other GPUs, vLLM versions, model revisions, or
  serving parameters.

## Operating Envelope

| Workload class                     |                                 Recommended operating point | Evidence                                                                                  | Production reading            |
| ---------------------------------- | ----------------------------------------------------------: | ----------------------------------------------------------------------------------------- | ----------------------------- |
| Interactive chat                   |     `<= 2 rps` offered load, `~1.94 completed rps` observed | p95 TTFT `541 ms`, p95 e2e `6.2s`, `100%` TTFT < `1s`                                     | GO                            |
| High-throughput internal assistant | around `5 rps` offered load, `~4.31 completed rps` observed | p95 TTFT `625 ms`, p95 e2e `12.1s`, `~2209 output tokens/s`                               | GO with monitoring            |
| Queueing boundary                  |                        between `5` and `6 rps` offered load | max inflight crosses the `max_num_seqs=64` region; p95 TTFT jumps from `625 ms` to `4.7s` | admission control recommended |
| Batch / agent jobs                 |                                     `8-10 rps` offered load | `0%` errors, but p95 TTFT `11.6-16.5s`                                                    | batch only                    |

## Main Finding: Capacity Frontier

![Open-loop capacity frontier](assets/open-loop-capacity-frontier.svg)

The deployment keeps sub-second p95 TTFT up to the `5 rps` offered-load
region. Above that point, output throughput increases only marginally
while TTFT rises sharply. This makes `5-6 rps` the practical boundary
between interactive/internal usage and batch-style serving for this
workload.

## Key Evidence

Source: [`data/request-rate-combined-summary.csv`](data/request-rate-combined-summary.csv)

| Offered RPS | Completed RPS | Max inflight | p95 TTFT |  p95 e2e | Server exact output TPS | Error rate | Reading                  |
| ----------: | ------------: | -----------: | -------: | -------: | ----------------------: | ---------: | ------------------------ |
|           2 |         1.936 |           13 |   541 ms |  6176 ms |                   991.5 |         0% | interactive              |
|           5 |         4.315 |           61 |   625 ms | 12073 ms |                  2209.2 |         0% | high-throughput internal |
|           6 |         4.588 |           89 |  4679 ms | 15495 ms |                  2349.0 |         0% | queueing begins          |
|           8 |         4.860 |          130 | 11563 ms | 20942 ms |                  2488.2 |         0% | batch leaning            |
|          10 |         4.938 |          167 | 16497 ms | 25948 ms |                  2528.0 |         0% | batch only               |

Important: this is not `10 rps` interactive serving. The benchmark
generated up to `10 rps` of offered load. Under this load, the deployment
completed approximately `4.94 requests/s` with `0%` errors and `0`
timeouts, but p95 TTFT reached `16.5s`.

## Goodput / SLO Pass Rate

Source: [`data/goodput-slo-summary.csv`](data/goodput-slo-summary.csv)

The table below is computed from per-request measured records after
removing warmup requests. The thresholds are illustrative production
SLOs, not universal latency requirements.

| Offered RPS | Completed RPS | Error-free | TTFT < 1s | TTFT < 5s | TPOT < 25ms | e2e < 15s | Reading                              |
| ----------: | ------------: | ---------: | --------: | --------: | ----------: | --------: | ------------------------------------ |
|           2 |         1.936 |       100% |      100% |      100% |        100% |      100% | interactive                          |
|           5 |         4.315 |       100% |      100% |      100% |        100% |      100% | high-throughput interactive/internal |
|           6 |         4.588 |       100% |    40.42% |      100% |      98.33% |    81.25% | queueing begins                      |
|           8 |         4.860 |       100% |    26.67% |    53.33% |        100% |     32.5% | batch leaning                        |
|          10 |         4.938 |       100% |    26.67% |    31.67% |        100% |    26.67% | batch only                           |

The most important diagnostic is that TTFT and e2e SLOs collapse before
TPOT does. The bottleneck is not per-token decode speed. The bottleneck
is queueing before first token once inflight requests exceed the serving
capacity region.

## Methodology

| Field                           | Value                                                                                                   |
| ------------------------------- | ------------------------------------------------------------------------------------------------------- |
| Run date                        | 2026-06-02                                                                                              |
| Benchmark mode                  | open-loop request-rate sweep                                                                            |
| Endpoint type                   | OpenAI-compatible streaming endpoint                                                                    |
| Workload                        | short prompt, long generation, decode-heavy                                                             |
| Model                           | `nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B-BF16`                                                            |
| Model snapshot                  | `cbd3fa9f933d55ef16a84236559f4ee2a0526848`                                                              |
| Max output tokens               | `512`                                                                                                   |
| Average generated output length | approximately `512` output tokens per completed request across the high-rate levels                     |
| Request-rate levels             | `0.25, 0.5, 0.75, 1, 1.25, 1.5, 2, 3, 4, 5, 6, 8, 10 rps`                                               |
| Measured requests               | `2310`                                                                                                  |
| Warmup                          | `3` warmup requests per benchmark run, excluded from measured totals                                    |
| Duration per level              | `120000 ms`                                                                                             |
| Cooldown between levels         | `60000 ms`                                                                                              |
| Sampling                        | `temperature=0`, `top_p=1`                                                                              |
| Reasoning mode                  | `enable_thinking=false`                                                                                 |
| Streaming                       | `true`                                                                                                  |
| Timeout                         | `120000 ms`                                                                                             |
| Client environment              | containerized benchmark runner on the same private serving path; private infrastructure details removed |
| Server runtime                  | vLLM `0.21.0`, PyTorch `2.11.0+cu130`, NVIDIA driver `595.58.03`, CUDA runtime `13.2`                   |
| Server hardware                 | 2 x NVIDIA RTX PRO 6000 Blackwell Server Edition, tensor parallel size `2`                              |

The public artifacts do not include a fixed random seed, `ignore_eos`,
client CPU/RAM allocation, or continuous GPU telemetry. Those are listed
as limitations and should be captured in the next benchmark run.

## Metric Definitions

| Metric                     | Definition                                                                                                              |
| -------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| Offered RPS / target RPS   | Rate at which the open-loop client initiates new requests. It is not the same as completed throughput under saturation. |
| Completed RPS / actual RPS | Completed measured requests divided by elapsed benchmark time for the level.                                            |
| TTFT                       | Time from request start until the first streamed token is received by the client.                                       |
| e2e latency                | Time from request start until the final token / completion is received by the client.                                   |
| ITL / TPOT                 | Inter-token latency, measured as average delay between successive output tokens during streaming.                       |
| Client output TPS          | Client-side output token estimate divided by benchmark duration.                                                        |
| Server exact output TPS    | Output token throughput from vLLM server-side token counters captured through metrics snapshots.                        |
| Error rate                 | HTTP/client errors plus timeouts divided by measured requests.                                                          |

## Server And vLLM Profile

The public manifest keeps only benchmark-relevant hardware and runtime
metadata. It removes private hostnames, paths, services, credentials, and
deployment details.

| Domain      | Public benchmark value                                                                |
| ----------- | ------------------------------------------------------------------------------------- |
| Runtime     | vLLM `0.21.0`, PyTorch `2.11.0+cu130`, NVIDIA driver `595.58.03`, CUDA runtime `13.2` |
| GPUs        | 2 x NVIDIA RTX PRO 6000 Blackwell Server Edition, 97887 MiB VRAM per GPU              |
| Parallelism | tensor parallel `2`, pipeline `1`, data parallel `1`                                  |
| vLLM limits | `max_model_len=32768`, `max_num_seqs=64`, `gpu_memory_utilization=0.90`               |
| Scheduler   | async scheduling and chunked prefill enabled                                          |
| CPU/RAM     | 32 logical CPUs, 251 GiB RAM                                                          |

`max_num_seqs=64` is the key interpretation threshold. TTFT stays below
1 second while observed inflight requests remain near or below this
limit. Once inflight requests exceed it substantially, vLLM keeps serving
without errors but introduces queueing before the first token.

## Charts

<details>
<summary>Additional open-loop charts</summary>

![Open-loop offered vs completed RPS](assets/open-loop-offered-vs-completed-rps.svg)

![Open-loop inflight vs p95 TTFT](assets/open-loop-inflight-vs-ttft.svg)

![Open-loop completed RPS vs p95 TTFT](assets/open-loop-rps-vs-ttft.svg)

</details>

<details>
<summary>Closed-loop throughput appendix charts</summary>

These charts come from a companion closed-loop throughput sweep. They
contextualize the throughput plateau, but the primary public result is
the open-loop request-rate sweep above.

![Concurrency vs p95 TTFT](assets/nemotron-load-p95-ttft.svg)

![Concurrency vs p95 e2e latency](assets/nemotron-load-p95-e2e.svg)

![Concurrency vs output TPS](assets/nemotron-load-output-tps.svg)

</details>

## Known Limitations

The request-rate workload is decode-heavy: each high-rate request used a
short prompt and `max_tokens=512`. Conclusions therefore apply most
directly to short-context, long-generation traffic. Long-document
workloads with `8k-30k` input tokens will have a different TTFT profile.

Current limitations:

- no continuous intra-level server scrape;
- no GPU telemetry in the public dataset;
- no 30-60 minute soak test;
- no mixed workload test combining chat, long-context, JSON, and batch;
- no multi-tenant autoscaling test;
- no long-context interactive latency benchmark;
- no cost/API parity analysis;
- no energy, temperature, clock, or throttling measurements;
- no fixed seed or client resource allocation captured in the public
  artifacts.

Near-32k long-context rejections in the broader benchmark were server
budget rejections, not model failures: `input_tokens + output_tokens`
exceeded the configured `max_model_len=32768`.

## Next Benchmark Roadmap

The next run should add continuous observability and richer production
coverage:

- scrape vLLM `/metrics` every `1-2s` during each level, including
  `num_requests_waiting`, `request_queue_time`, `kv_cache_usage_perc`,
  `num_preemptions`, TTFT histograms, TPOT/ITL, and e2e latency;
- collect GPU telemetry through DCGM or equivalent: utilization, VRAM
  used, power draw, temperature, clocks, throttling, and energy;
- add open-loop charts and tables for p99, not only p95;
- run a 30-60 minute soak at the recommended interactive and batch
  ceilings;
- add mixed workloads and long-context interactive cases;
- publish cost per million output tokens and watts per million output
  tokens if the underlying cost and power data can be shared.

## Data Files

| Path                                                                                                                           | Content                                                                                                                                |
| ------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------- |
| [`data/request-rate-combined-summary.csv`](data/request-rate-combined-summary.csv)                                             | Combined request-rate summary from `0.25` to `10 rps`                                                                                  |
| [`data/goodput-slo-summary.csv`](data/goodput-slo-summary.csv)                                                                 | Per-level SLO pass rates computed from measured request records                                                                        |
| [`data/baseline-request-rate/manifest.public.json`](data/baseline-request-rate/manifest.public.json)                           | Public manifest for `0.25` to `2 rps`                                                                                                  |
| [`data/baseline-request-rate/metrics.json`](data/baseline-request-rate/metrics.json)                                           | Aggregate benchmark metrics for baseline run                                                                                           |
| [`data/baseline-request-rate/stress-profile.csv`](data/baseline-request-rate/stress-profile.csv)                               | Per-level stress profile for baseline run                                                                                              |
| [`data/baseline-request-rate/stress-results.metrics-only.jsonl`](data/baseline-request-rate/stress-results.metrics-only.jsonl) | Per-request metrics with generated content removed                                                                                     |
| [`data/baseline-request-rate/server-metrics.public.jsonl`](data/baseline-request-rate/server-metrics.public.jsonl)             | Public server metric snapshots with endpoint removed                                                                                   |
| [`data/high-request-rate/manifest.public.json`](data/high-request-rate/manifest.public.json)                                   | Public manifest for `3` to `10 rps`                                                                                                    |
| [`data/high-request-rate/metrics.json`](data/high-request-rate/metrics.json)                                                   | Aggregate benchmark metrics for high-rate run                                                                                          |
| [`data/high-request-rate/stress-profile.csv`](data/high-request-rate/stress-profile.csv)                                       | Per-level stress profile for high-rate run                                                                                             |
| [`data/high-request-rate/stress-results.metrics-only.jsonl`](data/high-request-rate/stress-results.metrics-only.jsonl)         | Per-request metrics with generated content removed                                                                                     |
| [`data/high-request-rate/server-metrics.public.jsonl`](data/high-request-rate/server-metrics.public.jsonl)                     | Public server metric snapshots with endpoint removed                                                                                   |
| [`data/reference-runs/`](data/reference-runs/)                                                                                 | Sanitized metrics-only appendices for smoke, quality, long-context, stress, and closed-loop throughput runs cited by the source report |
| [`DATA_NOTICE.md`](DATA_NOTICE.md)                                                                                             | Sanitization and inclusion notice                                                                                                      |

The `reference-runs` directory is included for traceability. It is not
the primary request-rate score. Each run directory contains a public
manifest, aggregate metrics, stress profile when applicable, and JSONL
metrics-only records. Raw generated response bodies are not included.

## Efficiency Metrics

The public artifacts support these simple efficiency readings:

| Offered load | Server exact output TPS | GPUs | Output TPS/GPU | Completed RPS/GPU |
| -----------: | ----------------------: | ---: | -------------: | ----------------: |
|      `2 rps` |                 `991.5` |    2 |        `495.8` |           `0.968` |
|      `6 rps` |                `2349.0` |    2 |       `1174.5` |           `2.294` |
|     `10 rps` |                `2528.0` |    2 |       `1264.0` |           `2.469` |

Cost per million tokens and watts per million tokens are intentionally
not reported because the public export does not include GPU power,
energy, cloud pricing, or amortized hardware cost data.

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
