# Demo Script

## Before the session (about 30 minutes ahead)

1. `docker compose up -d` (about 1 to 2 minutes). Run `caffeinate -d` on macOS so the laptop stays awake.
2. Resolve all open cases, including risk override and settlement cases from earlier Demo 3 runs and rollout cases from Demo 4.
3. Pre-run: start `1_deploy_new_cert`. Flows 1 to 4 turn green.
4. Wait up to 5 minutes for the fresh freshness case. Check: description, SLA, linked asset `reference_data`, no case action attached.
5. Browser tabs: executions list, cases board filtered to Open, `2_rotate_certs` in the editor, `7_2_large_order` in the editor, `7_1_order_events` in the editor.
6. Run `8_0_load_fluss_reference` once, so the asset `fluss_reference_table` exists and the lineage shows the Fluss step.
7. Have a backup recording ready.

## Demo 1, the silent failure (8 minutes, pre-run)

| Time | Where | Talk track |
|---|---|---|
| 0:00 | Story slide | "Pre-market readiness. Overnight, the market data feed builds the reference data, such as the security master and corporate actions, that the trading desks need at the open. Four teams own four steps. In production these run hours apart. We compress the night into a minute." |
| 0:45 | Executions list (labels show domain and team) | "Four domains, four teams, one list. Every step succeeded: the rotation rotated, the restart restarted, the rebuild rebuilt. None of them knew what the step before actually delivered. Green means the task ran. It doesn't mean the outcome is right." |
| 1:45 | Freshness case | "A case is how Kestra tracks an incident: created automatically, assigned to the right team, with a priority and an SLA. Market data support knows what is broken, not why." |
| 2:45 | Linked asset → Dependencies → DAG | "You may know data lineage from ETL tools. Kestra extends it across your IT inventory. The certificate and the feed handler changed minutes ago. The reference data didn't: the rebuild ran green, received zero records and correctly published nothing." |
| 4:15 | `feed_certificate` → Executions → logs | "Two writers, one asset. The deploy wrote today's certificate. Half a second later, the rotation linked last week's. Both green." |
| 5:30 | `2_rotate_certs`: YAML, topology, no-code, Revisions | "Declarative YAML, not a proprietary language. Here is the cause: a hardcoded path, there since revision one. Nobody touched the logic." |
| 7:00 | Close | "Case: what. Lineage: where. Executions: when. YAML: why. Revisions: who. Four teams each see one box. Nobody sees the chain." |

## Demo 2, the agent analyzes and a human decides (4 minutes, live)

| Time | Where | Talk track |
|---|---|---|
| 0:00 | Agent slide | "Same incident. Finding the cause took four teams a morning. Now let an agent walk the chain." |
| 0:30 | Freshness case → Add action → `6_agent_root_cause_analysis` → run | "Any flow can become a case action. One cross-domain agent: it can read all four domains and open a case. It cannot change a single flow." |
| 1:15 | Agent execution → topology | "It reads each flow upstream, backwards through the chain. The model is one line of YAML: Gemini, Claude, OpenAI, or local and air-gapped." |
| 2:00 | Root cause case (PlatformAdmins) | "Four flows assessed, one flagged, the exact line, and a proposed diff. In seconds, not a morning." |
| 3:00 | Human decision, resolve with "Fix via Git, reviewed" | "Good first proposal. As the reviewer, I'd take the path from the deploy flow instead of recomputing the date. That's why a human keeps the decision." |
| 3:45 | Close | "Agents propose, humans decide. Nothing changes in production until a person acts." |

## Demo 3, from the order event to settlement (8 minutes, live)

Start `7_0_publish_order_events` at 0:30. It stands in for the Kafka producers and sends three events on `orders.workflow` to `7_1_order_events`: a standard order, a large order and the standard order a second time. The order ids are new on every run and end in `-standard` or `-large`, so rehearsals need no reset. Each execution of `7_1_order_events` carries the labels `order_id` and `order` (`standard`, `large` or `duplicate`). The large order starts `7_2_large_order`, which pauses at `risk_override` and opens a risk override case. To show the process alone, start `7_2_large_order` manually with its default inputs.

| Time | Where | Talk track |
|---|---|---|
| 0:00 | Overview slide | "The night is over and the reference data is fresh again. Now the desk trades. Orders arrive as events on Kafka. Fraud and limit checks per order run in Flink, the fast execution path stays in the order management system. Only orders from 250,000 USD land on the workflow topic." |
| 0:30 | `7_0_publish_order_events` → run, then the executions list with its labels | "Three events, three executions, one per event. In production this is a Kafka trigger. No consumer service to write, no offsets to manage." |
| 1:00 | `7_1_order_events` with `order:standard`, then `order:duplicate` | "The standard order, 400,000 USD, is checked and recorded and goes back to the OMS without approval. The third event is the same order again, as Kafka can deliver it after a restart. The order id is already in the KV store, so it stops. At-least-once delivery, exactly one trade." |
| 1:45 | `7_1_order_events` with `order:large`, then the `7_2_large_order` it started → topology | "A pension fund buys 50,000 shares at 50 dollars, 2.5 million in total. Above one million the order becomes a process that runs until settlement tomorrow. Three phases: pre-trade, execution, post-trade. Core banking, risk analytics and the trading platform each own a flow in their own namespace. This process belongs to trading operations and owns the chain." |
| 2:30 | Gantt, then the subflow execution of `7_3_reserve_buying_power` → logs | "Core banking and the risk check run in parallel. The core banking API timed out on the first attempt. Retry with exponential backoff, second attempt succeeds, and nobody got paged." |
| 3:00 | Logs of `call_pre_trade_check` | "In production a Flink job checks every order in milliseconds, against the reference data from the night. Here it finds a soft limit breach on concentration." |
| 3:30 | Cases board → risk override case → Acknowledge | "The approval is a case on the risk desk's board, with the reason and an SLA. Acknowledging stops the first SLA clock: the risk officer has seen it. The execution waits, and after 15 minutes without a decision the order fails and the buying power is released automatically." |
| 4:15 | Linked execution → Resume, `approved` true, comment | "The risk officer overrides the limit with a reason. Who decided, when and why is now part of the execution history." |
| 4:45 | Back to the case → Resolve, an existing reason plus the decision in the note | "The case closes with the decision on record." |
| 5:00 | Topology after the run, `execution` and `post_trade` phases | "The order is routed and filled. In production an algorithm fills it over the day, and settlement waits for the event on T+1. Here it takes seconds. Then it is booked and settled. The status goes out to Kafka at each step." |
| 5:45 | YAML of `7_2_large_order`, the `errors` blocks | "Error handling differs per phase. Before the trade we release the money. During execution we cancel the open order. After execution there is nothing to roll back, the trade is real, so a settlement fail opens an escalation case for operations." |
| 6:45 | `afterExecution` block, audit log line | "Whatever happens, the audit entry is written once the order has ended, with its final state." |
| 7:15 | Close | "Each team owns its step, the process owns the chain, from the Kafka event to settlement." |

### Which domain shows what in Demo 3

| Flow | Domain | Team | What it shows |
|---|---|---|---|
| `7_0_publish_order_events` | applications | trading-platform | a stand-in for the Kafka producers, only for the demo |
| `7_1_order_events` | applications | trading-platform | one execution per event, idempotency on the order id, a gateway by order size |
| `7_2_large_order` | business-process | trading-operations | the long-running process with phases, gateway, human override with timeout, compensation, escalation and audit |
| `7_3_reserve_buying_power` | applications | core-banking | a call to the core banking API with retry and exponential backoff |
| `7_4_pre_trade_check` | data | risk-analytics | the limit check, Flink in production, reading the Fluss reference table |
| `7_5_route_order` | applications | trading-platform | routing to the exchange and waiting for the fill with a timeout |

## Demo 4, changing the platform under the trading day (90 seconds, live)

Flink and Fluss run underneath Demo 3 as applications of the streaming platform team, and Kestra is not in their data path. Changing them is a process. The story is an emergency change: market hours are normally frozen, but since the open a faulty limit rule in the pre-trade check rejects valid orders, and the fix cannot wait for the night. Demo 4 deliberately looks like Demo 3. That is the point.

| Time | Where | Talk track |
|---|---|---|
| 0:00 | Operating Flink and Fluss slide | "Under the trading day runs the streaming platform. Flink checks every order, Fluss holds the reference data, a different team owns them. Since the open, a faulty limit rule rejects valid orders. The fix cannot wait for the night, so it is an emergency change." |
| 0:15 | `8_1_flink_job_rollout` → Execute, then Resume and approve | "A second person approves. The Flink Kubernetes Operator rolls out the new version with a savepoint, Kestra owns the change around it." |
| 0:40 | Topology after the run | "The new version is not healthy, so it goes back to the savepoint, and a case goes to the platform team. Approval, check, rollback, case. Looks familiar? That is the point." |
| 1:00 | Executions list of the last hour, unfiltered | "Infrastructure, applications, data, business process. Seven teams, one list, one audit trail. Unified orchestration, here you go." |
| 1:20 | Close | "In production the echo steps are the Kubernetes plugin and a call to the Flink REST API. The flow stays the same." |

Optional, if there is time: run `8_0_load_fluss_reference` and show Assets → `fluss_reference_table` → Dependencies, the lineage from the night of Demo 1 through Fluss into the pre-trade check.

### Which domain shows what in Demo 4

| Flow | Domain | Team | What it shows |
|---|---|---|---|
| `8_0_load_fluss_reference` | data | reference-data | the nightly load into a Fluss primary-key table, the lineage from `reference_data` to the pre-trade check |
| `8_1_flink_job_rollout` | infrastructure | streaming-platform | savepoint, approval during market hours, health probes, rollback as compensation, concurrency limit of one |

## Likely questions

- **Why isn't zero records an error?** On a quiet day or a holiday there may genuinely be no new corporate actions. The rebuild does its job; the freshness rule judges the outcome.
- **And if the certificate had been right?** Records arrive, the rebuild publishes, the freshness rule stays quiet. Rule: reference data must be republished within 30 minutes before the open.
- **Where's Kafka?** In production, the rebuild is triggered by the feed's refresh event on Kafka. In the demo it follows the restart. In Demo 3, a Kafka event on `orders.workflow` would start one execution per order, and the status goes out on `orders.status`.
- **Is the agent token read-only?** In production, run it under a read-only service account. RBAC decides what it can touch.
- **Is Kestra replacing Flink or the exchange connectivity?** No. Flink checks every order in milliseconds and the trading platform talks to the exchange. Kestra orchestrates the long-running process around them, which can last until the next day.
- **Why no rollback after execution?** Once the trade is executed, the shares are bought. You cannot undo a trade, you can only fix what follows, so a settlement fail goes to people as a case.
- **Why not let the Flink Kubernetes Operator do all of it?** The operator does the upgrade, savepoints and even an automatic rollback, and it should. It only knows its own job. The approval, the change record, the order across teams (schema first, then producer, then the Flink job), the business check and the audit trail sit around it.
- **Is a Flink job one task per order?** No. A streaming job runs 24/7 and every order flows through it. Kestra is never in that path. It orchestrates the lifecycle of the job and batch jobs that start and end.
- **Why not a real Kafka in the demo?** The orchestration is real, only the endpoints are mocked. Latency, throughput and consumer groups are what Kafka does, and the demo makes no claims about them.
- **One execution per event, even at high volume?** No. For hundreds of thousands of events per second, Flink filters and aggregates first and publishes a condensed signal, and the signal starts the flow.
- **Who can approve the override?** In the demo anyone who can resume the execution. In production the human task is assigned to the risk team via RBAC.
- **Enterprise Edition?** Assets, freshness triggers, cases and human tasks are EE features.

## Watch-outs

- On the agent task, open Input/Output, not Logs (harmless OpenTelemetry stack trace).
- Don't apply the fix live: flow 4 always receives zero records in the demo.
- In Demo 3, resume within 15 minutes, otherwise the pause fails the order on purpose. To show the rejection path instead, resume with `approved` set to false and point to `release_buying_power` in the `pre_trade` errors.
- A risk override case stays open until you resolve it. Resolve it after resuming, so the board is clean for the next run.
- In Demo 3 the standard order and the duplicate finish in seconds. Only the large order waits, at the risk override.
