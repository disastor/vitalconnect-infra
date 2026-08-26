# VitalConnect Demo — Mock Healthcare App for CloudBees Unify

A stand-in for VitalConnect's real environment, built to run the Aug 26 technical
discovery/demo call around one thesis: **can CloudBees untangle a homegrown flag
mess without a repo split or a rewrite?**

## What's here

Three services that mirror what Pamela described — fragmented flag sources, no
central visibility, and behavior that varies per hospital:

| Service | Role | Port | Stands in for |
|---|---|---|---|
| `config-relay` | Aggregates 3 fragmented legacy flag sources (DB JSON, env vars, YAML config) into one response. **This is the "before" picture** — it's what gets retired once Unify is wired in. | 4000 | Their current scattered flag storage |
| `vitalpatch-api` | Mock ECG/vitals telemetry. Response shape changes based on flags resolved via `flagClient.js`. | 4001 | VitalPatch device/telemetry service |
| `vista-center` | Hospital dashboard UI. Org + role switcher, card grid that changes per flag, a Flag Inspector panel, and an Incident Console that refreshes live from Unify after you change a flag there. | 8080 | Vista Center (the nurse-facing app) |

Three mock hospitals are seeded to hit every talking point from the call:

- **Sunrise General** (`hosp-sunrise`) — legacy tier, nothing rolled out yet
- **Lakeside Children's** (`hosp-lakeside`) — pilot cohort, `nurse-dashboard-v2` on, pediatric arrhythmia view
- **Metro Cardiac & MCT Center** (`hosp-metro`) — post-merger tier, MCT reporting workflow on — this is your "acquisition just doubled our scale" hospital

## The integration point that matters (now live)

`vista-center` and `vitalpatch-api` each connect to the **VitalConnect**
application in CloudBees Unify via the `rox-node` SDK, wrapped in their own
`flagClient.js`. `getFlags(orgId, role)` kept the same signature it had when
it called the fake `config-relay` aggregator — that's the "your code doesn't
change shape, only where the decision comes from" story made real, not just
asserted.

Flag names in code must exactly match the flag names created in Unify:
`nurse-dashboard-v2`, `hospital-specific-arrhythmia-view`,
`mct-reporting-workflow`. Targeting context (`organization_id`, `role`) is
passed per-call rather than set globally, since this is a multi-tenant
server evaluating flags for different hospitals concurrently.

**`config-relay` is retired from the live path.** Its code stays in the repo
as the explicit "before" reference — deliberately not deleted, so you can
point at it during the call and say "this is what your database/env-var/YAML
aggregation looks like today; here's what replaces it." It's excluded from
`docker-compose.yml` and has no k8s manifest anymore.

### Getting the SDK key into the app

1. In Unify: **Feature management → installation/SDK icon → SDK setup**, copy
   the key for the environment you're deploying (`vital_staging` or
   `vital_prod`).
2. Create a k8s Secret (don't put the key in a manifest file or commit it):
   ```bash
   kubectl create secret generic unify-sdk-key \
     --from-literal=sdk-key='<paste the environment's SDK key>' \
     -n vitalconnect-demo
   ```
3. Both `vista-center` and `vitalpatch-api` read it as `UNIFY_SDK_KEY` via
   `secretKeyRef` (see `k8s/*.yaml`).
4. For local testing, export it as an env var before `docker compose up`:
   ```bash
   export UNIFY_SDK_KEY='<key>'
   docker compose up --build
   ```

If you want both `vital_staging` and `vital_prod` running live simultaneously
(rather than just one for the call), you'll need two Secrets and two sets of
Deployments/Services in separate namespaces, since a single Secret/Deployment
pair can only point at one environment's key at a time.

## Run it locally first

```bash
docker compose up --build
```

Then open http://localhost:8080. Switch hospitals/roles in the left rail,
open the Flag Inspector to see live values pulled from Unify, and use the
Incident Console: change `nurse-dashboard-v2` or `mct-reporting-workflow`
for a hospital directly in Unify's UI, then click **Refresh from Unify** —
watch the waveform flatline and the new value take effect, no redeploy.

## Getting this into GitHub

You mentioned you'll create the repos — a couple of layout options:

- **Monorepo** (simplest for a demo): one repo `vitalconnect-demo` with these
  three folders as-is. Good if you want one PR/one Cloud Build trigger.
- **Split repos**: `vista-center`, `vitalpatch-api`, `config-relay` as separate
  repos, which more closely mirrors VitalConnect's real "flags spread across
  multiple teams and products" structure — arguably more on-brand for this
  specific demo, since the repo-per-team sprawl *is* part of the story.

Either works with the k8s manifests below unmodified — they only care about
image names, not repo layout.

## Deploying to your GCP cluster

Assuming a GKE cluster you already have `kubectl` access to, and the GCP CLI
authenticated (`gcloud auth login`, `gcloud config set project PROJECT_ID`):

```bash
# 1. One-time: create an Artifact Registry repo for the images
gcloud artifacts repositories create vitalconnect-demo \
  --repository-format=docker \
  --location=REGION

gcloud auth configure-docker REGION-docker.pkg.dev

# 2. Build and push each image (run from the repo root, or per-repo if split)
for svc in config-relay vitalpatch-api vista-center; do
  docker build -t REGION-docker.pkg.dev/PROJECT_ID/vitalconnect-demo/$svc:latest ./$svc
  docker push REGION-docker.pkg.dev/PROJECT_ID/vitalconnect-demo/$svc:latest
done

# 3. Point kubectl at your cluster
gcloud container clusters get-credentials YOUR_CLUSTER_NAME --region REGION

# 4. Apply the manifests (edit REGION/PROJECT_ID placeholders in k8s/*.yaml first)
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/config-relay.yaml
kubectl apply -f k8s/vitalpatch-api.yaml
kubectl apply -f k8s/vista-center.yaml

# 5. Get the external IP for the dashboard
kubectl get service vista-center -n vitalconnect-demo --watch
```

Once `EXTERNAL-IP` shows up, that's your demo URL. `config-relay` and
`vitalpatch-api` stay `ClusterIP` (internal only) — only `vista-center` needs
a public `LoadBalancer`.

If you'd rather avoid a public LoadBalancer for a customer-facing demo,
swap that Service to `ClusterIP` and use `kubectl port-forward
svc/vista-center -n vitalconnect-demo 8080:80` during the call instead.

## Mapping back to the call

| Demo moment | Where it lives |
|---|---|
| "Nobody knows what each flag is doing" | Flag Inspector panel — shows all 3 legacy sources per flag |
| "One codebase, different hospitals" | Switch org selector between Sunrise / Lakeside / Metro |
| "Production incident, disable without redeploy" | Incident Console → change the flag in Unify → Refresh from Unify → new value live, no rebuild |
| "Governance and audit" | Unify's own **Audit log** tab on the flag — real attribution, real timestamps, not a demo stand-in |
| "Migration without a rewrite" | `flagClient.js` — one function, one swap point |

## Status

Done: `VitalConnect` application created in Unify, `vital_staging` /
`vital_prod` environments linked, `vista-center` / `vitalpatch-api` /
`config-relay` components linked for code references, all three flags
created and targeted by `organization_id` (and `role` for
`nurse-dashboard-v2`), and both services now call Unify live via `rox-node`.

Still open:

- Create the `unify-sdk-key` k8s Secret and deploy to the cluster (see
  above) — do this at least a day before the call so you have time to debug
  a bad key or missed targeting rule, not during the walkthrough.
- Governance/RBAC: custom roles and an approval flow for production changes,
  so the "developers self-serve in staging, prod requires approval"
  beat has something real behind it, not just narration.
- Dry-run the whole sequence once end-to-end before Friday: switch hospitals,
  open the Flag Inspector, change a flag in Unify, hit Refresh, watch it
  update.
