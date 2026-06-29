# ss


## OpenMeter (usage metering, entitlements & billing)

  This chart bundles **OpenMeter** as a child dependency (declared in `Chart.yaml`, vendored under `charts/openmeter/`). OpenMeter is a usage-based **metering +
  billing** platform; deploying the chart brings up its full stack alongside the gateway.

  **Capabilities:**

  - **Metering** — real-time ingestion and aggregation of usage events into meters (COUNT/SUM/etc.).
  - **Entitlements** — usage-based access control: feature grants, quotas/limits, and live balances (e.g. "1M tokens/month included"), enforced via the
  `balance-worker`.
  - **Billing** — products, price plans, subscriptions, and invoice generation (with Stripe integration), via the `billing-worker`.
  - **Notifications** — balance/threshold alerts and webhook delivery via the `notification-service` + `svix`.

  **Components:** `openmeter-api` (ingest + query + entitlements/billing API), `sink-worker` (Kafka → ClickHouse), `balance-worker` (entitlements/balances),
  `billing-worker` (invoicing — enable via `config.billing`), `notification-service` + `svix` (webhooks), **ClickHouse** (usage store, via Altinity operator), plus
  its **Kafka / Postgres / Redis** backing deps.

  ### How it flows

  ```
  ingest:  producer → openmeter-api → Kafka → sink-worker → ClickHouse
  read:    usage / balances / invoices ← openmeter-api (ClickHouse + Postgres)
  notify:  threshold/balance events → notification-service → svix → webhooks
  ```

  - **AI Gateway integration:** after each LLM call, the gateway emits one CloudEvent to `http://openmeter-api.<ns>.svc.cluster.local/api/v1/events` with `subject` =
  tenant and `data` = model + token counts. Meters turn that into usage; entitlements enforce quotas; billing turns it into invoices.
  - **Data stores:** usage events → **ClickHouse**; meters, entitlements, balances, subscriptions, invoices, customers, and svix data → **Postgres**. Kafka =
  transit, Redis = dedup/cache. Back up ClickHouse + Postgres.

  ### Meters

  Defined in `values.yaml` under `openmeter.config.meters`. To add one: edit values + `helm upgrade`. Supported aggregations on this version: `COUNT, SUM, AVG, MIN,
  Beyond meters, OpenMeter manages **features → entitlements → plans → subscriptions → invoices**. Entitlements (grants/quotas) are computed by `balance-worker`;
  invoicing is produced by `billing-worker` (enable under `config.billing`, and connect a payment provider such as Stripe). These are managed via the OpenMeter API /
  portal, on top of the same metered usage.

  ### Dependency configuration (Kafka / Postgres / Redis)

  Each dep runs **bundled**, **in-cluster (other namespace)**, or on **Azure (managed)**:

  - `openmeter.<dep>.enabled: true` → bundled (chart auto-wires it)
  - `openmeter.<dep>.enabled: false` → external; set the endpoint under `openmeter.config.*`

  See [`OPENMETER-DEPS-GUIDE.txt`](./OPENMETER-DEPS-GUIDE.txt) for the full config matrix (exact keys + example values for all three modes per dependency).

  **svix** has its own Postgres DB + Redis. It follows the bundled deps by default; for external/Azure set `openmeter.svix.dbDSN` + `svix.redisDSN` **and** create a
  `svix` database + user on that Postgres.

  ### High availability

  - **Kafka (bundled):** `controller.replicaCount: 3`, `podAntiAffinityPreset: hard`, a PDB, and `extraConfig` with `default.replication.factor=3` +
  `min.insync.replicas=2` (otherwise auto-created topics are RF=1). Needs ≥3 schedulable nodes.
  - **Postgres / Redis (bundled):** only get replication, **not** automatic failover. For true HA use **CloudNativePG** / **Azure Flexible Server** (Postgres) and
  **Azure Cache** (Redis).

  ### Gotchas

  - Private GHCR images — create the `org-registry-cred` pull secret in the deploy namespace before installing.
  - Enabling a bundled Bitnami dep via `helm upgrade` (when it was off before) fails with a Bitnami **"PASSWORDS ERROR"** → uninstall + delete PVCs + reinstall.
  - On reinstall, the Altinity operator may race-delete the new ClickHouse CHI → re-apply: `helm template <rel> . -n <ns> | awk '/clickhouse.altinity.com/{p=1}
  p&&/^---/{p=0} p' | kubectl -n <ns> apply -f -`
