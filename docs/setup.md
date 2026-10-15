# Setup

Total time: about 15 minutes on a running Kestra EE instance. Demo 3 and Demo 4 need no additional setup beyond their flows.

## 1. Groups (IAM → Groups), about 2 minutes

| Group | Used by |
|---|---|
| `MarketDataSupport` | freshness case in `5_data_alerts` |
| `PlatformAdmins` | approval in `3_restart_feed`, root cause case in `6_agent_root_cause_analysis`, rollout approval in `8_1_flink_job_rollout` |

Add yourself as a member of both, so cases show up under "Assigned to me". Group names must match exactly.

Demo 3 needs no new group. Its cases are not assigned to a group, so you find them on the cases board filtered to Open. Demo 4 reuses `PlatformAdmins` for the approval of the Flink rollout.

## 2. Secrets, about 3 minutes

| Namespace | Secret | Purpose |
|---|---|---|
| `company.team` | `GEMINI_API_KEY` | AI agent model provider |
| `company.team` | `KESTRA_API_TOKEN` | agent reads flows via the Kestra API |
| `data` | `API_KEY` | API token injected by the plugin policy into all `io.kestra.plugin.kestra.ee` tasks (cases) |

Create API tokens via your profile (bottom left). Token values are shown only once.
In production, use a read-only service account for the agent.

Demo 3 needs no new secret. The plugin policy in `docker-compose.yml` reads `API_KEY` from the `data` namespace explicitly, so it also covers the cases that `7_2_large_order` creates in `apps.trading`.

## 3. Flows, about 5 minutes

Create the flows from `flows/` in their namespaces, in order 1 to 6, then the Demo 3 and Demo 4 flows. Each file declares its own namespace. A flow must exist before the flows that call it, so for Demo 3 create them backwards: `7_5_route_order`, `7_4_pre_trade_check` and `7_3_reserve_buying_power`, then `7_2_large_order`, `7_1_order_events` and `7_0_publish_order_events`. For Demo 4 create `8_0_load_fluss_reference` and `8_1_flink_job_rollout`.

## 4. Smoke test, about 5 minutes

1. Start `1_deploy_new_cert`. Flows 1 → 2 → 3 → 4 should run green within about 30 seconds.
2. In flow 4, `skip_publish` runs and `publish_reference_data` is skipped.
3. Assets → `reference_data` → Dependencies → DAG shows the full chain.
4. Within 5 minutes (once `reference_data` is older than 30 minutes), `5_data_alerts` opens a critical case for `MarketDataSupport`.
5. Start `7_0_publish_order_events`. Three executions of `7_1_order_events` run: the standard order goes back to the OMS without approval, the large order starts `7_2_large_order`, and the redelivered standard order logs that it was already processed.
6. The `7_2_large_order` execution pauses at `risk_override` and opens a risk override case. Resume with `approved` set to true. The execution ends in success, with the audit entry in the logs. Resolve the risk override case afterwards.
7. Run `8_0_load_fluss_reference`. Assets → `fluss_reference_table` → Dependencies shows `reference_data` upstream and `7_4_pre_trade_check` downstream.
8. Start `8_1_flink_job_rollout` with its default inputs and approve the human task. The health check stays unhealthy after three probes, `rollback_to_savepoint` runs once and a case opens.

## Notes

- **Kafka and Postgres are not required.** Flow 4 keeps its Kafka trigger commented out and follows the restart instead. Flow `7_1_order_events` keeps its Kafka trigger commented out too, and `7_0_publish_order_events` sends the order events instead.
- **The approval gate in flow 3** is controlled by `MARKET_OPEN: false` in `3_restart_feed`, which simulates an overnight rotation without approval. Set it to `true` to show the human approval step.
- **Demo 1, Demo 3 and Demo 4 run offline** once the images are pulled. Demo 2 needs internet for the model API.
- **The agent API path** assumes tenant `main` and `http://localhost:8080`. Test with
  `curl -H "Authorization: Bearer <TOKEN>" "http://localhost:8080/api/v1/main/flows/infra/2_rotate_certs?source=true"` (about 1 second).
