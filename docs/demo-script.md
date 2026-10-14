# Demo Script

## Before the session (about 30 minutes ahead)

1. `docker compose up -d` (about 1 to 2 minutes). Run `caffeinate -d` on macOS so the laptop stays awake.
2. Resolve all open cases, including risk override and settlement cases from earlier Demo 3 runs.
3. Pre-run: start `1_deploy_new_cert`. Flows 1 to 4 turn green.
4. Wait up to 5 minutes for the fresh freshness case. Check: description, SLA, linked asset `reference_data`, no case action attached.
5. Browser tabs: executions list, cases board filtered to Open, `2_rotate_certs` in the editor, `7_0_large_order` in the editor.
6. Have a backup recording ready.

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

## Demo 3, one large order across three teams (7 minutes, live)

Start `7_0_large_order` with the default inputs at 0:30. The order breaches the soft limit, so the execution pauses at `risk_override` and a risk override case opens. Resume with `approved` set to true, and the rest of the run takes about 10 seconds.

| Time | Where | Talk track |
|---|---|---|
| 0:00 | Overview slide | "The night is over and the reference data is fresh again. Now the desk trades. A pension fund buys 50,000 shares at a limit of 50 dollars, 2.5 million in total. That is not one API call, it is a transaction that runs until settlement tomorrow." |
| 0:30 | `7_0_large_order` → Execute with defaults → topology | "Three phases: pre-trade, execution, post-trade. Core banking, risk analytics and the trading platform each own a flow in their own namespace. This process belongs to trading operations and owns the chain." |
| 1:15 | Execution → Gantt, then the subflow execution of `7_1_reserve_buying_power` → logs | "Core banking and the risk check run in parallel. The core banking API timed out on the first attempt. Retry with exponential backoff, second attempt succeeds, and nobody got paged." |
| 2:00 | Logs of `call_pre_trade_check`, then the asset `reference_data` | "In production a Flink job checks every order in milliseconds. It reads the reference data from Demo 1. If that data had been stale this morning, every limit check would have run on yesterday's numbers." |
| 2:45 | Cases board → risk override case | "Soft limit breach on concentration. The approval is a case on the risk desk's board, with the reason and an SLA. The execution waits, and after 15 minutes without a decision the order fails and the buying power is released automatically." |
| 3:45 | Execution → Resume, `approved` true, comment, then resolve the case | "The risk officer overrides the limit with a reason. Who decided, when and why is now part of the execution history." |
| 4:30 | Topology after the run, `execution` and `post_trade` phases | "The order is routed and filled. In production an algorithm fills it over the day, and settlement waits for the event on T+1. Here it takes seconds. Then it is booked and settled. The status goes out to Kafka at each step." |
| 5:15 | YAML of `7_0_large_order`, the `errors` blocks | "Error handling differs per phase. Before the trade we release the money. During execution we cancel the open order. After execution there is nothing to roll back, the trade is real, so a settlement fail opens an escalation case for operations." |
| 6:15 | `finally` block, audit log line | "Whatever happens, the audit entry is written. Approved, rejected, timed out, settled." |
| 6:40 | Close | "Each team owns its step, the process owns the chain. Same platform as last night's incident, same lineage, same cases." |

### Which domain shows what in Demo 3

| Flow | Domain | Team | What it shows |
|---|---|---|---|
| `7_0_large_order` | business-process | trading-operations | the long-running process with phases, gateway, human override with timeout, compensation, escalation and audit |
| `7_1_reserve_buying_power` | applications | core-banking | a call to the core banking API with retry and exponential backoff |
| `7_2_pre_trade_check` | data | risk-analytics | the limit check, Flink in production, reading the `reference_data` asset |
| `7_3_route_order` | applications | trading-platform | routing to the exchange and waiting for the fill with a timeout |

## Likely questions

- **Why isn't zero records an error?** On a quiet day or a holiday there may genuinely be no new corporate actions. The rebuild does its job; the freshness rule judges the outcome.
- **And if the certificate had been right?** Records arrive, the rebuild publishes, the freshness rule stays quiet. Rule: reference data must be republished within 30 minutes before the open.
- **Where's Kafka?** In production, the rebuild is triggered by the feed's refresh event on Kafka. In the demo it follows the restart. In Demo 3, a Kafka event on `orders.requested` would start one execution per order, and the status goes out on `orders.status`.
- **Is the agent token read-only?** In production, run it under a read-only service account. RBAC decides what it can touch.
- **Is Kestra replacing Flink or the exchange connectivity?** No. Flink checks every order in milliseconds and the trading platform talks to the exchange. Kestra orchestrates the long-running process around them, which can last until the next day.
- **Why no rollback after execution?** Once the trade is executed, the shares are bought. You cannot undo a trade, you can only fix what follows, so a settlement fail goes to people as a case.
- **Who can approve the override?** In the demo anyone who can resume the execution. In production the human task is assigned to the risk team via RBAC.
- **Enterprise Edition?** Assets, freshness triggers, cases and human tasks are EE features.

## Watch-outs

- On the agent task, open Input/Output, not Logs (harmless OpenTelemetry stack trace).
- Don't apply the fix live: flow 4 always receives zero records in the demo.
- In Demo 3, resume within 15 minutes, otherwise the pause fails the order on purpose. To show the rejection path instead, resume with `approved` set to false and point to `release_buying_power` in the `pre_trade` errors.
- A risk override case stays open until you resolve it. Resolve it after resuming, so the board is clean for the next run.
