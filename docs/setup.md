# Setup

Total time: about 15 minutes on a running Kestra EE instance.

## 1. Groups (IAM → Groups), about 2 minutes

| Group | Used by |
|---|---|
| `MarketDataSupport` | freshness case in `5_data_alerts` |
| `PlatformAdmins` | approval in `3_restart_feed`, root cause case in `6_agent_root_cause_analysis` |

Add yourself as a member of both, so cases show up under "Assigned to me". Group names must match exactly.

## 2. Secrets, about 3 minutes

| Namespace | Secret | Purpose |
|---|---|---|
| `company.team` | `GEMINI_API_KEY` | AI agent model provider |
| `company.team` | `KESTRA_API_TOKEN` | agent reads flows via the Kestra API |
| `data` | `API_KEY` | API token injected by the plugin policy into all `io.kestra.plugin.kestra.ee` tasks (cases) |

Create API tokens via your profile (bottom left). Token values are shown only once.
In production, use a read-only service account for the agent.

## 3. Flows, about 5 minutes

Create the flows from `flows/` in their namespaces, in order 1 to 6. Each file declares its own namespace.

## 4. Smoke test, about 5 minutes

1. Start `1_deploy_new_cert`. Flows 1 → 2 → 3 → 4 should run green within about 30 seconds.
2. In flow 4, `skip_publish` runs and `publish_reference_data` is skipped.
3. Assets → `reference_data` → Dependencies → DAG shows the full chain.
4. Within 5 minutes (once `reference_data` is older than 30 minutes), `5_data_alerts` opens a critical case for `MarketDataSupport`.

## Notes

- **Kafka and Postgres are not required.** Flow 4 keeps its Kafka trigger commented out and follows the restart instead.
- **Approval gate:** `MARKET_OPEN: false` in `3_restart_feed` simulates an overnight rotation without approval. Set it to `true` to show the human approval step.
- **Offline:** Demo 1 runs offline once the images are pulled. Demo 2 needs internet for the model API.
- **Agent API path:** the agent assumes tenant `main` and `http://localhost:8080`. Test with
  `curl -H "Authorization: Bearer <TOKEN>" "http://localhost:8080/api/v1/main/flows/infra/2_rotate_certs?source=true"` (about 1 second).
