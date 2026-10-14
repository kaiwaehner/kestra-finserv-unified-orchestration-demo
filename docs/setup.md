# Setup

Total time: about 15 minutes on a running Kestra EE instance. Demo 3 needs no additional setup beyond its four flows.

## 1. Groups (IAM → Groups), about 2 minutes

| Group | Used by |
|---|---|
| `MarketDataSupport` | freshness case in `5_data_alerts` |
| `PlatformAdmins` | approval in `3_restart_feed`, root cause case in `6_agent_root_cause_analysis` |

Add yourself as a member of both, so cases show up under "Assigned to me". Group names must match exactly.

Demo 3 needs no new group. Its cases are not assigned to a group, so you find them on the cases board filtered to Open.

## 2. Secrets, about 3 minutes

| Namespace | Secret | Purpose |
|---|---|---|
| `company.team` | `GEMINI_API_KEY` | AI agent model provider |
| `company.team` | `KESTRA_API_TOKEN` | agent reads flows via the Kestra API |
| `data` | `API_KEY` | API token injected by the plugin policy into all `io.kestra.plugin.kestra.ee` tasks (cases) |

Create API tokens via your profile (bottom left). Token values are shown only once.
In production, use a read-only service account for the agent.

Demo 3 needs no new secret. The plugin policy in `docker-compose.yml` reads `API_KEY` from the `data` namespace explicitly, so it also covers the cases that `7_0_large_order` creates in `apps.trading`.

## 3. Flows, about 5 minutes

Create the flows from `flows/` in their namespaces, in order 1 to 6, then the Demo 3 flows. Each file declares its own namespace. Create the three subflows `7_1_reserve_buying_power`, `7_2_pre_trade_check` and `7_3_route_order` before `7_0_large_order`, which calls them.

## 4. Smoke test, about 5 minutes

1. Start `1_deploy_new_cert`. Flows 1 → 2 → 3 → 4 should run green within about 30 seconds.
2. In flow 4, `skip_publish` runs and `publish_reference_data` is skipped.
3. Assets → `reference_data` → Dependencies → DAG shows the full chain.
4. Within 5 minutes (once `reference_data` is older than 30 minutes), `5_data_alerts` opens a critical case for `MarketDataSupport`.
5. Start `7_0_large_order` with the default inputs. It pauses at `risk_override` and opens a risk override case. Resume with `approved` set to true. The execution ends in success, with the audit entry in the logs. Resolve the risk override case afterwards.

## Notes

- **Kafka and Postgres are not required.** Flow 4 keeps its Kafka trigger commented out and follows the restart instead. Flow `7_0_large_order` keeps its Kafka trigger commented out too and starts manually.
- **The approval gate in flow 3** is controlled by `MARKET_OPEN: false` in `3_restart_feed`, which simulates an overnight rotation without approval. Set it to `true` to show the human approval step.
- **Demo 1 and Demo 3 run offline** once the images are pulled. Demo 2 needs internet for the model API.
- **The agent API path** assumes tenant `main` and `http://localhost:8080`. Test with
  `curl -H "Authorization: Bearer <TOKEN>" "http://localhost:8080/api/v1/main/flows/infra/2_rotate_certs?source=true"` (about 1 second).
