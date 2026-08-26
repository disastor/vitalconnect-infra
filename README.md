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
| `vista-center` | Hospital dashboard UI. Org + role switcher, card grid that changes per flag, a Flag Inspector panel, and an Incident Console that toggles a flag live and records an audit entry. | 8080 | Vista Center (the nurse-facing app) |

Three mock hospitals are seeded to hit every talking point from the call:

- **Sunrise General** (`hosp-sunrise`) — legacy tier, nothing rolled out yet
- **Lakeside Children's** (`hosp-lakeside`) — pilot cohort, `nurse-dashboard-v2` on, pediatric arrhythmia view
- **Metro Cardiac & MCT Center** (`hosp-metro`) — post-merger tier, MCT reporting workflow on — this is your "acquisition just doubled our scale" hospital

## The integration point that matters

Every service reaches flags through one function: `getFlags(orgId, role)` in
`flagClient.js` (see `vitalpatch-api/flagClient.js`). Today it calls
`config-relay`. When you wire in the real CloudBees Unify SDK, **this is the
only place that changes** — swap the `fetch()` call for `client.variation()` /
`boolVariation()`, keep the same signature, and every caller stays untouched.
That's the "your code doesn't change, only where the decision comes from"
story Pamela needs to believe.

Once that swap happens, `config-relay` itself becomes unnecessary — worth
showing that retirement explicitly in the demo as step 9 of the migration
story (inventory → pilot flag → SDK in → validate → cut over → repeat →
remove old logic).

## Run it locally first

```bash
docker compose up --build
```

Then open http://localhost:8080. Switch hospitals/roles in the left rail,
open the Flag Inspector to see the fragmented "sources" per flag, and use the
Incident Console to toggle `nurse-dashboard-v2` or `mct-reporting-workflow`
for a hospital — watch the waveform flatline and the audit entry appear.

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
| "Production incident, disable without redeploy" | Incident Console → toggle → audit entry, no rebuild |
| "Governance and audit" | Audit log entries (today: no real attribution — deliberately crude, to contrast with what Unify provides) |
| "Migration without a rewrite" | `flagClient.js` — one function, one swap point |

## Next steps once you're in Unify

You mentioned you'll handle wiring the CloudBees pieces in yourself — worth
creating, per the demo narrative:

- Applications/environments matching `vista-center`, `vitalpatch-api`
- Flags: `nurse-dashboard-v2` (boolean), `hospital-specific-arrhythmia-view`
  (multivariate: standard / pediatric / cardiac-mct), `mct-reporting-workflow`
  (boolean)
- Targeting rules on `organization_id` and `role` context attributes
- A custom role/approval flow for production changes, to show off the
  governance story alongside the crude in-memory audit log this repo ships with
