---
layout: post
title: "SLOs, SLIs, and Error Budgets"
date: 2026-03-15 09:20:00 +0530
categories: observability sre
tags: [slo, sli, error-budget, sre, prometheus, reliability]
---

# SLOs, SLIs, and Error Budgets

Uptime dashboards that say **99.9%** without defining **what was measured** are fan fiction. **SLIs** are the measurements. **SLOs** are the targets you commit to. **Error budgets** are the remainder—the only honest language for “can we ship this risky change?”

This post ties SLIs to Prometheus-style metrics, multi-window burn alerts, and how gateways vs services should own different SLOs.

---

## Definitions

**SLI (Service Level Indicator).** A quantitative measure of user-visible reliability.

Examples:

- Availability: `successful_requests / total_requests` (define success)
- Latency: fraction of requests faster than 300 ms
- Freshness: async pipeline lag < 60 s
- Correctness: (harder) sampled comparisons, not just HTTP 200

**SLO (Service Level Objective).** The target: 99.9% of requests succeed over 30 days.

**SLA.** The **contract** with customers (credits, legal). Usually looser than the SLO you run internally. Do not alert on the SLA; alert **before** you burn the SLO.

**Error budget.** `1 - SLO`. At 99.9% over 30 days you may “spend” 0.1% failed requests. If you spend it early, **freeze risky deploys** or invest in reliability. If you never spend it, you might be **too slow** to ship (or the SLO is too weak).

---

## Good SLIs Are User-Centric

Bad: CPU < 80%. Users do not feel CPU.

Better: **checkout POST success ratio** and **checkout latency at p99** as measured at the **API gateway** (or client RUM), not only inside a happy pod.

Internal services still need SLIs—but they are **dependency SLIs**. A payments SLI of 99.99% may be required so checkout can hit 99.9%.

**Window.** 30 days is common for product SLOs. Alerting uses **shorter** windows so you do not wait 29 days to notice a break.

---

## Measuring with Prometheus

Availability SLI:

```promql
sum(rate(http_requests_total{job="gateway", route="/checkout", code!~"5.."}[5m]))
/
sum(rate(http_requests_total{job="gateway", route="/checkout"}[5m]))
```

Decide: are **4xx** failures? Usually **not** for availability (client bugs). **401** from your auth outage might be. Document it.

Latency SLI with histogram:

```promql
sum(rate(http_request_duration_seconds_bucket{route="/checkout", le="0.3"}[5m]))
/
sum(rate(http_request_duration_seconds_count{route="/checkout"}[5m]))
```

This is “good seconds / total”—the SRE workbook approach. Averages hide the tail; **histograms + `le`** match the SLO.

---

## Multi-Window, Multi-Burn Alerts

A single “error rate > 1% for 5 minutes” either **pages constantly** or **never pages**.

Google’s **burn rate** idea: if you are burning budget **14×** on a 1-hour window **and** still burning on a 5-minute window, page. If you are burning slowly over 6 hours, ticket.

You need:

- Long window (detect significant budget spend)
- Short window (ensure the issue is still happening)

Alert on **SLI**, not on “deployment happened.” Correlate deploys on the dashboard.

---

## Where to Place the SLI

| Layer | Use |
|-------|-----|
| **RUM / mobile** | Closest to user; includes CDN and last mile |
| **API gateway** | Best **server-side** user SLI for APIs |
| **Service mesh** | Great for **dependency** SLIs (success between A→B) |
| **App RED** | Implementation detail; still needed for debugging |

Do not average 40 microservice SLIs into one “platform SLO.” Users hit **journeys** (login, search, checkout). Map journeys to gateway routes or BFF operations.

---

## Error Budget Policy (Social, Not Technical)

Without a written policy, SLO is a Grafana panel.

A light policy:

1. Budget **healthy** → ship as usual.
2. Budget **50% consumed** with 2 weeks left → extra review on risky changes.
3. Budget **exhausted** → reliability work takes priority over features until a window recovers (or you **renegotiate** the SLO with product).

The last point is important: a chronically exhausted budget means the **SLO is a lie** or the **system cannot meet it**. Fix product expectations or architecture.

---

## Load Tests and SLOs

A load test that only reports RPS is incomplete. Report **SLI during the test**: did latency SLI stay above the SLO at target RPS? Gate releases on that, not on “we hit 10k RPS once.”

---

## Takeaways

SLIs measure **user journeys**. SLOs are **targets with windows**. Error budgets are **how you decide speed vs stability**. Measure at the **gateway or RUM**, alert with **burn rates**, and keep mesh/app metrics for **diagnosis**.

If every service is “five nines” on internal HTTP 200s while checkout fails at the edge, you have **vanity SLIs**. Move the measurement to where the user is.

---

✍️ *Written by Chirag Gupta — documenting my journey in Go Concurrency & Cloud Native.*
