# Demo Script

## Before the session (about 30 minutes ahead)

1. `docker compose up -d` (about 1 to 2 minutes). Run `caffeinate -d` on macOS so the laptop stays awake.
2. Resolve all open cases.
3. Pre-run: start `1_deploy_new_cert`. Flows 1 to 4 turn green.
4. Wait up to 5 minutes for the fresh freshness case. Check: description, SLA, linked asset `reference_data`, no case action attached.
5. Browser tabs: executions list, cases board filtered to Open, `2_rotate_certs` in the editor.
6. Have a backup recording ready.

## Demo 1 (8 minutes, pre-run): the silent failure

| Time | Where | Talk track |
|---|---|---|
| 0:00 | Story slide | "Pre-market readiness. Overnight, the market data feed builds the reference data, such as the security master and corporate actions, that the trading desks need at the open. Four teams own four steps. In production these run hours apart. We compress the night into a minute." |
| 0:45 | Executions list (labels show domain and team) | "Four domains, four teams, one list. Every step succeeded: the rotation rotated, the restart restarted, the rebuild rebuilt. None of them knew what the step before actually delivered. Green means the task ran. It doesn't mean the outcome is right." |
| 1:45 | Freshness case | "A case is how Kestra tracks an incident: created automatically, assigned to the right team, with a priority and an SLA. Market data support knows what is broken, not why." |
| 2:45 | Linked asset → Dependencies → DAG | "You may know data lineage from ETL tools. Kestra extends it across your IT inventory. The certificate and the feed handler changed minutes ago. The reference data didn't: the rebuild ran green, received zero records and correctly published nothing." |
| 4:15 | `feed_certificate` → Executions → logs | "Two writers, one asset. The deploy wrote today's certificate. Half a second later, the rotation linked last week's. Both green." |
| 5:30 | `2_rotate_certs`: YAML, topology, no-code, Revisions | "Declarative YAML, not a proprietary language. Here is the cause: a hardcoded path, there since revision one. Nobody touched the logic." |
| 7:00 | Close | "Case: what. Lineage: where. Executions: when. YAML: why. Revisions: who. Four teams each see one box. Nobody sees the chain." |

## Demo 2 (4 minutes, live): the agent analyzes, a human decides

| Time | Where | Talk track |
|---|---|---|
| 0:00 | Agent slide | "Same incident. Finding the cause took four teams a morning. Now let an agent walk the chain." |
| 0:30 | Freshness case → Add action → `6_agent_root_cause_analysis` → run | "Any flow can become a case action. One cross-domain agent: it can read all four domains and open a case. It cannot change a single flow." |
| 1:15 | Agent execution → topology | "It reads each flow upstream, backwards through the chain. The model is one line of YAML: Gemini, Claude, OpenAI, or local and air-gapped." |
| 2:00 | Root cause case (PlatformAdmins) | "Four flows assessed, one flagged, the exact line, and a proposed diff. In seconds, not a morning." |
| 3:00 | Human decision, resolve with "Fix via Git, reviewed" | "Good first proposal. As the reviewer, I'd take the path from the deploy flow instead of recomputing the date. That's why a human keeps the decision." |
| 3:45 | Close | "Agents propose, humans decide. Nothing changes in production until a person acts." |

## Likely questions

- **Why isn't zero records an error?** On a quiet day or a holiday there may genuinely be no new corporate actions. The rebuild does its job; the freshness rule judges the outcome.
- **And if the certificate had been right?** Records arrive, the rebuild publishes, the freshness rule stays quiet. Rule: reference data must be republished within 30 minutes before the open.
- **Where's Kafka?** In production, the rebuild is triggered by the feed's refresh event on Kafka. In the demo it follows the restart.
- **Is the agent token read-only?** In production, run it under a read-only service account. RBAC decides what it can touch.
- **Enterprise Edition?** Assets, freshness triggers, cases and human tasks are EE features.

## Watch-outs

- On the agent task, open Input/Output, not Logs (harmless OpenTelemetry stack trace).
- Don't apply the fix live: flow 4 always receives zero records in the demo.
