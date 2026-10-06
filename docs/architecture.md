# Architecture (V1)

## Flow chain and agent

Five flows owned by four teams run as a chain. Every flow succeeds, yet the reference data is stale. Only the freshness check in flow 5 looks at the outcome and opens a case. From that case, the cross-domain agent (flow 6) walks the chain backwards, finds the root cause and opens a second case with a proposed fix. A human reviews and decides; the code fix goes through Git.

![V1 overview: the incident chain across four domains, and the AI agent walking it backwards](images/v1-overview.svg)

## Asset lineage

Assets connect the flows. No flow calls another flow's code; each flow only declares what it reads and writes. Kestra builds the lineage graph from these declarations, which is what the analyst (Demo 1) and the agent (Demo 2) follow backwards.

```mermaid
flowchart TB
  F1["1_deploy_new_cert"] -- writes --> AC[("feed_certificate<br/>File")]
  F2["2_rotate_certs"] -- writes --> AC
  AC -- read by --> F3["3_restart_feed"]
  F3 -- writes --> AF[("feedhandler<br/>External")]
  AF -- read by --> F4["4_update_reference_data"]
  F4 -. "writes only if new records" .-> AR[("reference_data<br/>Dataset")]
  AR -- "watched by freshness trigger" --> F5["5_data_alerts"]
```

## How the agent investigates

```mermaid
sequenceDiagram
  participant Case as Freshness case
  participant Agent as Flow 6: agent
  participant API as Kestra API (read only)
  participant LLM as Model (Gemini)
  participant RCA as Root cause case
  Case->>Agent: case action, stale_asset = reference_data
  Agent->>API: read flows 4, 3, 2, 1 (walking backwards)
  API-->>Agent: flow definitions (YAML)
  Agent->>LLM: flows + feed handler log excerpt
  LLM-->>Agent: root cause, evidence, proposed diff
  Agent->>RCA: open case for PlatformAdmins
  Note over RCA: A human reviews the proposal.<br/>Nothing changes until a person acts.
```

## What is simulated

All external systems are simulated with `echo` commands: the certificate store, the feed handler, the vendor connection and the reference data build. The feed handler log line in the agent prompt is a fixed excerpt. Kafka and Postgres are not required; flow 4 follows the restart instead of a Kafka event.
