# SAS Viya Logging (OpenSearch) on OpenShift 4.20 — Internal Test Guide, Air-Gapped From the Start

Project version: 1.2.54. Scope: **log monitoring** (`logging/bin/deploy_logging_openshift.sh`). The metric side (Grafana) is in `OPENSHIFT_MONITORING_INTERNAL_TEST.md`.

Method: no "connected first" stage. Every image and Helm chart is mirrored into Harbor (project `monitoring`) and the deployment runs with `AIRGAP_DEPLOYMENT=true`, exactly as the client will do with Quay. Podman is the container tool.

Everything below comes from reading the project's scripts and values files. **Nothing has been run yet**; the lab log in section 11 is empty on purpose.

---

## 0. What gets deployed

One namespace, `logging` (`LOG_NS`). `deploy_logging_openshift.sh` runs these steps in order:

| Step | What | Notes |
|---|---|---|
| 0 | OpenShift prerequisites | Grants the **privileged** SCC to service account `v4m-os`; creates SCCs `v4m-logging-v2` and `v4m-k8sevents` (Fluent Bit reads host log folders) |
| 1 | OpenSearch (StatefulSet `v4m-search`) | Default **3 nodes, 4 GiB Java heap each, 30 GiB volume each**. Waits up to 10 min for the pod, then 2 min more, then runs the security initialisation |
| 2 | OpenSearch Dashboards (`v4m-osd`) | The web UI |
| 3 | Elasticsearch metrics exporter | Feeds OpenSearch metrics to Prometheus |
| 4 | OpenSearch content | Index templates, ingest pipelines, index-retention policies (through the OpenSearch API) |
| 5 | Dashboards content | Index patterns, saved searches, dashboards (through the OSD API; waits up to about 8 min) |
| 6 | Fluent Bit (DaemonSet `v4m-fb`) | Collects container logs from **every node and every namespace**, including OpenShift's own |
| 7 | Fluent Bit for Kubernetes events | Second, small Fluent Bit workload |
| 8 | Route `v4m-osd` | `https://v4m-osd-logging.apps.<domain>` |
| 9 | ServiceMonitors | Lets OpenShift's Prometheus scrape OpenSearch and Fluent Bit |
| 10 | Version info | A small local Helm release |

The API steps (4 and 5) reach OpenSearch and Dashboards with `kubectl port-forward` from the bastion (`LOG_ALWAYS_PORT_FORWARD` defaults to `true`), so no route is needed for them. The bastion needs `curl`.

### 0.1 Images and charts needed

Paths follow the same rule as the metric side: `<source registry>/<repo>/<image>:<tag>` is stored as `$REG/<repo>/<image>:<tag>`.

| Component | Source image | Stored at |
|---|---|---|
| OpenSearch | `docker.io/opensearchproject/opensearch:3.6.0` | `$REG/opensearchproject/opensearch:3.6.0` |
| OpenSearch Dashboards | `docker.io/opensearchproject/opensearch-dashboards:3.6.0` | `$REG/opensearchproject/opensearch-dashboards:3.6.0` |
| Metrics exporter | `quay.io/prometheuscommunity/elasticsearch-exporter:v1.10.0` | `$REG/prometheuscommunity/elasticsearch-exporter:v1.10.0` |
| Fluent Bit | `cr.fluentbit.io/fluent/fluent-bit:5.0.7` | `$REG/fluent/fluent-bit:5.0.7` |
| Init containers (OpenSearch sysctl, Fluent Bit) | `docker.io/library/busybox:latest` | `$REG/library/busybox:latest` |

| Chart | Version | Stored at |
|---|---|---|
| `opensearch/opensearch` | 3.6.0 | `oci://$REG/opensearch` |
| `opensearch/opensearch-dashboards` | 3.6.0 | `oci://$REG/opensearch` |
| `prometheus-community/prometheus-elasticsearch-exporter` | 7.2.1 | `oci://$REG/prometheus-community` |
| `fluent/fluent-bit` | 0.57.7 | `oci://$REG/fluent` |

Two things to notice:
- The Fluent Bit **image and chart share one repository path**, `monitoring/fluent/fluent-bit` (image tag `5.0.7`, chart tag `0.57.7`). Harbor should accept that. Section 3.4 tests it, because Quay might not.
- `busybox:latest` is used twice and cannot be pinned without overriding two variables. Record its digest (3.2).

### 0.2 Optional: logs inside Grafana

Needs the metric deployment (Grafana) and one extra file, the OpenSearch datasource plugin zip. Section 9.

---

## 1. Decisions to make before you start

### 1.1 Storage: this is the main risk in your lab

Your lab's only storage class is `truenas-nfs`, and it is now the default class, so OpenSearch would use it. **Lucene, the engine inside OpenSearch, is not supported on NFS.** Red Hat's own guidance for OpenShift Elasticsearch says NFS-type storage is not supported because Lucene relies on file-system behaviour NFS does not provide, and that data corruption and other problems can occur; block storage is recommended ([Red Hat article](https://access.redhat.com/articles/6988737)). OpenSearch is built on the same engine.

So:
- **Best:** use a block storage class for OpenSearch (an iSCSI class if your TrueNAS CSI driver offers one, Ceph RBD, LVM Storage on a node with a spare disk, or similar). Put its name in `persistence.storageClass` (section 4.3). Ask your TrueNAS admin what `csi.truenas.io` can provide.
- **Acceptable for a first smoke test only:** try on `truenas-nfs` and expect possible failures (lock errors, `AccessDenied`, slowness, corruption after a restart). Do not judge the deployment or the client design by an NFS result.

The **client** must have real block storage for this component; add that to the client's checklist.

### 1.2 Size for the lab

Defaults (`logging/opensearch/opensearch_helm_values.yaml`): 3 replicas, `-Xms4096m -Xmx4096m`, 30 GiB each. That is about 12 GiB of Java heap alone, and pods use roughly twice their heap. Unless your workers are large, shrink it for the lab (section 4.3):

| | Default | Lab, 1 node | Lab, 2 nodes |
|---|---|---|---|
| `replicas` | 3 | 1 | 2 |
| `opensearchJavaOpts` | `-Xms4096m -Xmx4096m` | `-Xms1g -Xmx1g` | `-Xms1g -Xmx1g` |
| `persistence.size` | 30Gi | 10Gi | 10Gi |
| Cluster health you should see | green | **yellow** (the project's index templates ask for 1 replica copy, which has no second node to live on; this is expected) | green |

Fluent Bit ships **all** logs, including OpenShift's own, so a 10 GiB volume can fill quickly. The retention policies delete old indices (`INFRA_LOG_RETENTION_PERIOD` defaults to 1 day for OpenShift infrastructure logs); watch the disk during the test.

### 1.3 OpenShift's own logging must not run at the same time

SAS warns that running OpenShift's optional logging stack next to this one duplicates DaemonSets and doubles storage. Check (section 5).

### 1.4 Privileged pods

Two things need elevated rights, and the client's security team should know:
- OpenSearch pods run a **privileged init container** (busybox) to set `vm.max_map_count` on the node. The project binds SCC `privileged` to service account `v4m-os` for that.
- Fluent Bit pods are **privileged** and mount host folders, through the custom SCC `v4m-logging-v2`.

The alternative to the first one is to set `vm.max_map_count=262144` on the nodes yourself (MachineConfig or Tuned) and set `sysctlInit: {enabled: false}` in the user values; the sample file describes both approaches. It is not needed for the lab.

### 1.5 The Route's certificate authority

Step 8 creates the Dashboards Route as **re-encrypt** and only adds an annotation (`cert-utils-operator.redhat-cop.io/destinationCA-from-secret`) that tells the Red Hat *cert-utils operator* to fill in the certificate authority. If that operator is not installed, the Route has **no destination CA**, OpenShift's router does not trust Dashboards' certificate (which comes from the project's own CA), and the Route returns "Application is not available". Section 7 fixes it by hand without any operator. This is likely to matter at the client, where installing an operator in an air gap means mirroring an operator catalog.

---

## 2. Preparation on the bastion

Same host and settings as the metric deployment. New tool: **`curl`** (the API steps use it).

```bash
cd /home/melsa/viya4-monitoring-kubernetes-1.2.54
export USER_DIR=$HOME/v4m-user

export REG_HOST=harbor.lab.datascience.me
export REG=$REG_HOST/monitoring
export REG_USER='robot$<account-with-push-rights>'
read -rs -p 'Harbor password: ' REG_PASS; export REG_PASS; echo

curl --version | head -1; oc whoami; oc auth can-i create namespace --all-namespaces
echo "$REG_PASS" | podman login $REG_HOST -u "$REG_USER" --password-stdin
echo "$REG_PASS" | helm registry login $REG_HOST -u "$REG_USER" --password-stdin
```
`REG_USER` must be able to **push** to project `monitoring` (a `..._pull_secret` robot usually cannot). Never write the password in a file.

---

## 3. Mirror everything into Harbor

Working folder: `mkdir -p ~/v4m-logging && cd ~/v4m-logging`

### 3.1 Images

```bash
cat > images-logging.txt <<'EOF'
docker.io/opensearchproject/opensearch:3.6.0
docker.io/opensearchproject/opensearch-dashboards:3.6.0
quay.io/prometheuscommunity/elasticsearch-exporter:v1.10.0
cr.fluentbit.io/fluent/fluent-bit:5.0.7
docker.io/library/busybox:latest
EOF

while read -r img; do podman pull --arch amd64 "$img"; done < images-logging.txt

while read -r img; do
  target="$REG/${img#*/}"          # drop the source host, keep <repo>/<image>:<tag>
  podman tag  "$img" "$target"
  podman push "$target"
done < images-logging.txt
```
`--arch amd64` assumes x86_64 nodes; check with `oc get nodes -o jsonpath='{.items[*].status.nodeInfo.architecture}'`.

### 3.2 Record what you mirrored
```bash
while read -r img; do echo "$img  $(podman image inspect --format '{{.Digest}}' "$img")"; done < images-logging.txt | tee images-logging-digests.txt
```

### 3.3 Charts
```bash
helm repo add opensearch          https://opensearch-project.github.io/helm-charts
helm repo add fluent              https://fluent.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

mkdir -p charts
helm pull opensearch/opensearch                                  --version 3.6.0  --destination charts
helm pull opensearch/opensearch-dashboards                       --version 3.6.0  --destination charts
helm pull prometheus-community/prometheus-elasticsearch-exporter --version 7.2.1  --destination charts
helm pull fluent/fluent-bit                                      --version 0.57.7 --destination charts

helm push charts/opensearch-3.6.0.tgz                          oci://$REG/opensearch
helm push charts/opensearch-dashboards-3.6.0.tgz               oci://$REG/opensearch
helm push charts/prometheus-elasticsearch-exporter-7.2.1.tgz   oci://$REG/prometheus-community
helm push charts/fluent-bit-0.57.7.tgz                         oci://$REG/fluent
```

### 3.4 Check everything reads back
```bash
helm show chart oci://$REG/opensearch/opensearch                                --version 3.6.0  | head -3
helm show chart oci://$REG/opensearch/opensearch-dashboards                     --version 3.6.0  | head -3
helm show chart oci://$REG/prometheus-community/prometheus-elasticsearch-exporter --version 7.2.1 | head -3
helm show chart oci://$REG/fluent/fluent-bit                                    --version 0.57.7 | head -3

while read -r img; do t="$REG/${img#*/}"; podman rmi "$t" >/dev/null; podman pull "$t" >/dev/null && echo "OK  $t"; done < images-logging.txt
```
Every chart must print its name and version, and every image line must say `OK`. Pay attention to `fluent/fluent-bit`: the chart (`0.57.7`) and the image (`5.0.7`) are in the same repository. If **either** read fails there, push the chart somewhere else: set `AIRGAP_HELM_REPO=<host>/monitoring/charts` **only** in `$USER_DIR/logging/user.env` (it is loaded only by the logging scripts, so the Grafana chart path is unaffected), push all four logging charts to `oci://$REG/charts/<repo>` instead, and repeat the check.

---

## 4. Namespace, pull secret and settings

### 4.1 Namespace and pull secret

The pull secret must exist **before** the deploy script runs (`airgap-include.sh` checks it and stops otherwise):
```bash
oc get ns logging || oc create ns logging

oc create secret docker-registry v4m-image-pull-secret -n logging \
  --docker-server=$REG_HOST --docker-username="$REG_USER" --docker-password="$REG_PASS"
```
Use the registry **host**, not `host/monitoring`. A pull-only robot is better than the push account here.

### 4.2 Settings folder

`USER_DIR` already contains the air-gap settings from the metric deployment (`AIRGAP_DEPLOYMENT=true`, `AIRGAP_REGISTRY=harbor.lab.datascience.me/monitoring`, `AIRGAP_IMAGE_PULL_SECRET_NAME=v4m-image-pull-secret`, `AIRGAP_HELM_FORMAT=oci`). Check them, and copy the logging templates:
```bash
grep -E '^AIRGAP_' $USER_DIR/user.env

mkdir -p $USER_DIR/logging
cp samples/generic-base/logging/user.env                  $USER_DIR/logging/
cp samples/generic-base/logging/user-values-opensearch.yaml $USER_DIR/logging/
```
Every line in those copies is commented out, so nothing changes until you edit them. Use `=` (never `:`) in `.env` files.

### 4.3 OpenSearch size and storage (`$USER_DIR/logging/user-values-opensearch.yaml`)

Uncomment and set (lab, 2 nodes; use `replicas: 1` if you only have room for one):
```yaml
replicas: 2
opensearchJavaOpts: "-Xms1g -Xmx1g"
persistence:
  enabled: true
  storageClass: <block-storage-class>     # delete this line only to accept the default class (NFS in your lab; see 1.1)
  size: 10Gi
```
Do **not** set `image:` keys; the scripts generate them.

### 4.4 Passwords (optional)

If you do nothing, the script generates random passwords and prints the `admin` one at the end. To choose them, set them in the shell for the deployment only, not in a file:
```bash
read -rs -p 'OpenSearch admin password: ' ES_ADMIN_PASSWD; export ES_ADMIN_PASSWD; echo
read -rs -p 'logadm password: '            LOG_LOGADM_PASSWD; export LOG_LOGADM_PASSWD; echo
```
Afterwards, change the `admin` password **only** with `logging/bin/change_internal_password.sh admin <new-password>`. The script's own notice says never to change it in the Dashboards web UI.

---

## 5. Pre-flight checks

```bash
# 1. OpenShift's own logging must not be running (each command should show nothing useful)
oc get clusterlogging -A 2>&1 | head -3
oc get ns openshift-logging 2>&1 | head -2
oc get pods -A 2>/dev/null | grep -Ei 'vector|fluentd|logging-collector|loki|openshift-logging' | head

# 2. Capacity: allocatable CPU and memory per node, and current use
oc get nodes -o custom-columns=NAME:.metadata.name,CPU:.status.allocatable.cpu,MEM:.status.allocatable.memory
oc adm top nodes

# 3. Storage classes (see 1.1)
oc get sc

# 4. Is the cert-utils operator installed? (no output = not installed; see 1.5 and section 7)
oc get csv -A 2>/dev/null | grep -i cert-utils

# 5. Tools
curl --version | head -1 ; yq --version ; helm version --short
```
Also confirm the metric deployment's user workload monitoring is still running: `oc get pods -n openshift-user-workload-monitoring` (step 9 creates ServiceMonitors that need it).

---

## 6. Deploy

```bash
cd /home/melsa/viya4-monitoring-kubernetes-1.2.54
export USER_DIR=$HOME/v4m-user
echo "USER_DIR=$USER_DIR"

logging/bin/deploy_logging_openshift.sh 2>&1 | tee ~/v4m-logging-deploy.log
```
Expect **15 to 30 minutes**. Most of it is waiting: OpenSearch (up to 10 min, then a fixed 2 min pause) and Dashboards (up to about 8 min).

**Check in the first lines of output:**
- `INFO User directory: /home/melsa/v4m-user`
- `Deploying into an 'air-gapped' cluster from private registry [harbor.lab.datascience.me/monitoring]`
- no `export: ... not a valid identifier` line

**Progress lines** (`STEP 0` to `STEP 10`):
- `STEP 0: OpenShift Setup`: SCCs `v4m-logging-v2` and `v4m-k8sevents` created.
- `STEP 1`: `Waiting on OpenSearch pods to be Ready`, then the `run_securityadmin.log` output, ending with `Done with success`. A warning `There may have been a problem with the run_securityadmin.log script` means it did not; see the troubleshooting table.
- In step 1 the Helm output should include a line like `Pulled: harbor.lab.datascience.me/monitoring/opensearch/opensearch:3.6.0`.
- `STEP 5`: `Waiting (up to more 8 minutes) for OpenSearch Dashboards API endpoint to be ready`.
- `STEP 8`: `OpenShift Route [v4m-osd] has been created.`

**At the end** the script prints the URL and, if it generated them, the passwords:
```
**The OpenSearch 'admin' Account**
Generated 'admin' password:  ...
```
Save that output (it is also in `~/v4m-logging-deploy.log`). To read the passwords later:
```bash
oc get secret -n logging internal-user-admin -o jsonpath='{.data.password}' | base64 -d; echo
```

---

## 7. Fix the Dashboards Route (when the cert-utils operator is absent)

Only if the pre-flight check 4 showed nothing. Test first: open `https://v4m-osd-logging.apps.ocp.lab.datascience.me`. If you get "Application is not available", do this:
```bash
oc -n logging get secret kibana-tls-secret -o jsonpath='{.data.ca\.crt}' | base64 -d > /tmp/osd-ca.crt
test -s /tmp/osd-ca.crt && head -1 /tmp/osd-ca.crt          # must show -----BEGIN CERTIFICATE-----

HOST=$(oc -n logging get route v4m-osd -o jsonpath='{.spec.host}')
oc -n logging delete route v4m-osd
oc -n logging create route reencrypt v4m-osd --service v4m-osd --port=http \
  --insecure-policy=Redirect --hostname "$HOST" --dest-ca-cert=/tmp/osd-ca.crt

oc -n logging get route v4m-osd
```
Re-running the deploy script later does not undo this: `create_openshift_route.sh` skips a route that already exists.

---

## 8. Verify

### 8.1 Workloads, volumes and images
```bash
oc get pods,pvc,svc,route -n logging
oc get sts,deploy,ds -n logging
```
- `v4m-search-0` (and `-1` if 2 replicas): `1/1 Running`; each PVC `Bound`.
- `v4m-osd-...` and the exporter: `Running`. One `v4m-fb-...` Fluent Bit pod **per node**, plus the events workload.

Every image must come from Harbor:
```bash
oc get pods -n logging -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.initContainers[*].image}{" | "}{.spec.containers[*].image}{"\n"}{end}'
```
Every image (init containers included) must begin with `harbor.lab.datascience.me/monitoring/`.

### 8.2 OpenSearch health
```bash
ADMIN_PW=$(oc get secret -n logging internal-user-admin -o jsonpath='{.data.password}' | base64 -d)
oc -n logging port-forward svc/v4m-search 9200:9200 >/dev/null 2>&1 &  PF=$!; sleep 4
curl -sk -u admin:"$ADMIN_PW" 'https://localhost:9200/_cluster/health?pretty'
curl -sk -u admin:"$ADMIN_PW" 'https://localhost:9200/_cat/indices?v'
kill $PF
```
Health: `green` (2 or 3 nodes) or `yellow` (1 node; expected). `red` is a failure. After a few minutes `_cat/indices` should list indices starting with `viya_`, including the OpenShift infrastructure one (`viya_logs-openshift-...`), which shows Fluent Bit is delivering.

### 8.3 Dashboards
1. Open the Route (`https://v4m-osd-logging.apps.ocp.lab.datascience.me`). A certificate warning is normal (self-signed ingress certificate). Log in as `admin` with the password from step 6.
2. **Discover**: pick the index pattern for the logs (the deployment creates several; the OpenShift infrastructure one matches `viya_logs-openshift-*`). Log lines must appear and keep arriving.
3. Search for messages from a pod you know, for example `openshift-monitoring` or `v4m-grafana`.
4. Your lab has no Viya, so the `viya_logs-*` indices for Viya pods stay empty. That is normal.

### 8.4 Metrics about logging (optional)
```bash
oc get servicemonitor,podmonitor -n logging
```
and, in Grafana (metric deployment), query `elasticsearch_cluster_health_status`.

**The deployment passes when:** 8.1 (all Running, all images from Harbor), 8.2 (not red, `viya_` indices exist) and 8.3 (log lines in Discover) succeed.

---

## 9. Optional: logs inside Grafana

Needs the metric deployment running in `monitoring`. In an air gap the plugin zip must be supplied by you (the script would otherwise use Grafana's online plugin installer).

```bash
# On a connected machine (your bastion has internet in the lab):
curl -fL -o grafana-opensearch-datasource-2.34.3.linux_amd64.zip \
  https://github.com/grafana/opensearch-datasource/releases/download/v2.34.3/grafana-opensearch-datasource-2.34.3.linux_amd64.zip
cp grafana-opensearch-datasource-2.34.3.linux_amd64.zip $USER_DIR/monitoring/

cd /home/melsa/viya4-monitoring-kubernetes-1.2.54 && export USER_DIR=$HOME/v4m-user
monitoring/bin/create_logging_datasource.sh
```
The script copies the zip into the running Grafana pod (`/var/lib/grafana/plugins`, on the Grafana volume), unzips it, creates the datasource secret, deploys the log-enabled dashboards, and restarts the Grafana pod. Check: Grafana → **Connections → Data sources** now lists an OpenSearch datasource; **Save & test** succeeds.

If the metric side uses OpenShift login, sign in again after the restart.

---

## 10. Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `The image pull secret ... was not detected` | Secret missing in `logging` (4.1) |
| `AIRGAP_REGISTRY has not been set` / wrong `User directory` | `USER_DIR` not exported |
| `oc adm policy ... privileged` fails | Not cluster-admin |
| `v4m-search-0` `Pending` | PVC unbound (`oc get pvc,events -n logging`), not enough CPU/memory, or a storage class problem |
| `v4m-search-0` `CrashLoopBackOff`, log mentions `max virtual memory areas vm.max_map_count` | The privileged sysctl init container did not run. Check `oc get pod v4m-search-0 -n logging -o yaml \| grep -i scc` and `oc logs v4m-search-0 -n logging --all-containers` |
| OpenSearch errors such as `failed to obtain node locks`, `AccessDeniedException`, `FileSystemException`, or very slow starts | The volume is NFS or not writable by OpenShift's UID: see 1.1. Use block storage |
| `ImagePullBackOff`, `manifest unknown` | Path mismatch: compare `oc describe pod` with the "Stored at" column in 0.1 |
| `ImagePullBackOff`, `x509` | The cluster does not trust Harbor's CA (metric guide, B2) |
| `ImagePullBackOff`, `unauthorized` | Wrong pull-secret credentials, or the account has no Read on `monitoring` |
| Helm cannot pull a chart | Not pushed to `oci://$REG/<repo>` (3.3/3.4); for `fluent-bit` see the note at the end of 3.4 |
| Warning about `run_securityadmin.log` | OpenSearch was not fully up. Re-run: `oc exec -n logging v4m-search-0 -c opensearch -- config/run_securityadmin.sh` and read its output |
| Step 5: "OpenSearch Dashboards API endpoint has NOT become accessible" | Dashboards is still starting (it can take several minutes) or cannot reach OpenSearch. `oc logs -n logging deploy/v4m-osd`, then re-run the deploy script |
| Route: "Application is not available" | Section 7 |
| Route hostname does not resolve | DNS for `*.apps.ocp.lab.datascience.me` |
| Fluent Bit pod `CreateContainerConfigError` / cannot start | SCC missing: `oc get scc v4m-logging-v2` |
| No `viya_logs-openshift-*` index | Fluent Bit cannot reach OpenSearch or authenticate: `oc logs -n logging ds/v4m-fb` |
| Cluster health `yellow` with 1 node | Expected (1.2) |
| Volume fills up | Raise `persistence.size`, or shorten `INFRA_LOG_RETENTION_PERIOD` / `LOG_RETENTION_PERIOD` in `$USER_DIR/logging/user.env` |

To start over:
```bash
LOG_DELETE_PVCS_ON_REMOVE=true logging/bin/remove_logging_openshift.sh
```
It removes the components, the Route, the custom SCCs, the ServiceMonitors, and (with the setting above) the OpenSearch volumes. Delete the volumes too when you redeploy: by default the script also deletes the generated password secrets, so a redeploy on old volumes would carry a different password than the data expects. The namespace and your `v4m-image-pull-secret` stay (that secret is not labeled as project-managed).

---

## 11. Lab log

| Date | Check | Result | Consequence |
|---|---|---|---|
| — | Storage class for OpenSearch (block vs `truenas-nfs`) | *pending* | |
| — | OpenShift Logging not installed | *pending* | |
| — | cert-utils operator installed? | *pending* | If not: section 7 is required (and matters at the client) |
| — | Mirror: 5 images + 4 charts in Harbor, read back (3.4) | *pending* | `fluent/fluent-bit` image + chart in one repo: *pending* |
| — | Deploy `deploy_logging_openshift.sh` (air-gap) | *pending* | |
| — | All images (incl. init) from Harbor | *pending* | |
| — | OpenSearch health, `viya_` indices | *pending* | |
| — | Dashboards Route works (with or without section 7) | *pending* | |
| — | Logs visible in Discover | *pending* | |
| — | Grafana logging datasource (section 9) | *pending* | |

## 12. When you are done

Send me the result of each step (or the first failing message and the relevant part of `~/v4m-logging-deploy.log`). Then I will write the client runbook for logging, in the same structure as the metric one: Quay on port 8443, the transfer bundle, the storage decision, the Route certificate fix, and anything that broke here.
