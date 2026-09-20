# SAS Viya Monitoring on OpenShift 4.20 — Internal Test Guide (Metrics / Grafana)

Project version: 1.2.54. Scope: **metric monitoring only** (`deploy_monitoring_openshift.sh`).
Logging (OpenSearch) is a separate deployment and is not covered here.

## What gets deployed

On OpenShift this project does **not** install Prometheus. It deploys only:

- **Grafana** (Helm chart `grafana-community/grafana` 12.10.4), a PVC of 5Gi
- **oauth-proxy sidecar** in the Grafana pod (OpenShift login for Grafana)
- A `grafana-serviceaccount` with `cluster-monitoring-view`, so Grafana can query OpenShift's Thanos Querier
- SAS Viya dashboards, recording rules, and a `v4m-grafana` **Route**

Metrics come from OpenShift's built-in monitoring + **user workload monitoring**.

Optional: `deploy_monitoring_viya.sh` adds a Pushgateway + ServiceMonitors/PodMonitors to the Viya namespace.

### Images and charts for this deployment

| Item | Source | Needed |
|---|---|---|
| Grafana image | `docker.io/grafana/grafana:13.0.3` | always |
| Sidecar image | `quay.io/kiwigrid/k8s-sidecar:2.10.1` | always |
| oauth-proxy image | `registry.redhat.io/openshift4/ose-oauth-proxy:latest` | always (default auth) |
| Grafana chart | `grafana-community/grafana` 12.10.4 | always |
| Pushgateway image + chart | `quay.io/prometheus/pushgateway:v1.11.3`, `prometheus-community/prometheus-pushgateway` 3.7.0 | only for `deploy_monitoring_viya.sh` |
| Tempo image + chart | `docker.io/grafana/tempo:2.10.7`, `grafana-community/tempo` 2.2.3 | only if `TRACING_ENABLE=true` |

## Plan: two stages

- **Stage A — connected deploy.** No registry involved. Proves the scripts, cluster permissions, storage, OAuth and Route all work.
- **Stage B — Harbor rehearsal.** Same deployment, but every image and chart comes from Harbor with `AIRGAP_DEPLOYMENT=true`. This is the rehearsal of the client's Quay flow.

Do Stage A first. If Stage B fails, you then know the problem is registry/paths, not the cluster.

---

## 0. Prepare the deployment host

The scripts are **bash**. Use a Linux host or WSL2 (Ubuntu) that can reach the OpenShift API and Harbor.

| Tool | Required version | Notes |
|---|---|---|
| bash | any | |
| `oc` | **4.19 or newer** (script rule: server minor − 1) | Download from the console: `?` → Command Line Tools. `kubectl` is in the same tarball |
| `kubectl` | 1.27+ | |
| `helm` | 3.8+ (v3 or v4) | OCI support needs 3.8+ |
| `yq` (mikefarah) | **4.45.1+** | Not the Python `yq` |
| `podman` | any recent | Stage B (lab and client both use Podman). `skopeo` is optional (only to list tags). Only `bin/setup_airgap.sh` insists on a command named `docker`, and this guide does not use that script; if you ever run it, install `podman-docker` (RHEL) so a `docker` command exists |
| `openssl`, `sha256sum` | any | |

Check:
```bash
oc version; helm version --short; yq --version; kubectl version --client
```

### Copy the project and fix permissions

Zip extraction on Windows drops the executable bit. From the repo root on the Linux host:
```bash
chmod +x bin/*.sh monitoring/bin/*.sh logging/bin/*.sh
# line endings must be LF (should print nothing):
grep -rIl $'\r' --include=*.sh . | head
```
If files show up, run `dos2unix` on them.

### Log in as cluster-admin
```bash
oc login https://api.<cluster-domain>:6443 -u kubeadmin      # or an admin token
oc whoami
oc auth can-i create namespace --all-namespaces              # must print: yes
```
The scripts refuse to run without cluster-admin.

---

## 1. Pre-flight checks on the cluster

```bash
# 1. Default storage class (Grafana needs a 5Gi PVC)
oc get sc
# One class must show "(default)". If none does, see "No default storage class" below.

# 2. Console route domain (the script derives the route domain from it)
oc get route -n openshift-console console -o jsonpath='{.spec.host}{"\n"}'
# Expected: console-openshift-console.apps.<your-domain>
# The script strips the first 26 characters. If yours looks different, set OPENSHIFT_ROUTE_DOMAIN yourself (step 2).

# 3. User workload monitoring
cd /home/melsa/viya4-monitoring-kubernetes-1.2.54
oc apply -f monitoring/openshift/cluster-monitoring-config.yaml    # wait for prometheus-user-workload etc. to be Running
oc -n openshift-monitoring get cm cluster-monitoring-config -o yaml
```
- **ConfigMap not found:** the script creates it with `enableUserWorkload: true`. Nothing to do.
- **ConfigMap exists:** the script does **not** change it. Make sure `config.yaml` contains `enableUserWorkload: true`:
  ```bash
  oc -n openshift-monitoring edit cm cluster-monitoring-config
  ```
- After it is enabled, this must list running pods (`prometheus-user-workload`, `thanos-ruler-user-workload`, `prometheus-operator`):
  ```bash
  oc get pods -n openshift-user-workload-monitoring
  ```

### No default storage class
The script only **warns** when there is no default class. The Grafana PVC then stays `Pending` and the Helm `--atomic` install times out and rolls back. Fix it before deploying, with one of:

**Option 1 — mark a class as default** (used in the internal lab; also covers the Pushgateway and OpenSearch PVCs later):
```bash
oc patch storageclass <name> -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
oc get sc        # must now show "(default)"
```

**Option 2 — leave the cluster default alone.** In `$USER_DIR/monitoring/user-values-openshift-grafana.yaml` (copied from `samples/generic-base` in step 2), uncomment `persistence:` and set `storageClassName`, so it contains:
```yaml
persistence:
  storageClassName: <your-storage-class>
```
The Pushgateway (`deploy_monitoring_viya.sh`) has its own setting in that case (`$USER_DIR/monitoring/user-values-pushgateway.yaml`, key `persistentVolume.storageClass`).

### Storage write test (do this before the first deploy)
A Bound PVC does not prove the pod can write to it. Grafana runs under OpenShift's restricted SCC with a random UID, so an NFS-backed class can bind fine and still fail with `Permission denied`. This test reproduces that situation:
```bash
oc create ns pvc-test
cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: t, namespace: pvc-test}
spec:
  accessModes: [ReadWriteOnce]
  resources: {requests: {storage: 1Gi}}
---
apiVersion: v1
kind: Pod
metadata: {name: t, namespace: pvc-test}
spec:
  containers:
  - name: t
    image: registry.access.redhat.com/ubi9/ubi-minimal
    command: ["sh","-c","touch /data/x && echo WRITE_OK; sleep 3600"]
    volumeMounts: [{name: d, mountPath: /data}]
  volumes: [{name: d, persistentVolumeClaim: {claimName: t}}]
EOF
oc get pvc,pod -n pvc-test        # PVC Bound, pod Running
oc logs -n pvc-test t             # expect: WRITE_OK
```
Clean up. With reclaim policy `Retain` the PV is left behind:
```bash
oc delete ns pvc-test
oc get pv        # remove the leftover pvc-test PV: oc delete pv <name>
```
`Permission denied` here means Grafana will fail the same way — fix the storage backend (dataset permissions / the CSI driver's fsGroup handling) before continuing.

(On the client cluster, the image for this test must come from their registry, or pick any image already mirrored there.)

### Enable user workload monitoring ahead of the script (optional but tidy)
If `cluster-monitoring-config` does not exist, the script would create it and wait only 30 seconds. Doing it first avoids a "pods not detected" warning:
```bash
oc apply -f monitoring/openshift/cluster-monitoring-config.yaml
oc get pods -n openshift-user-workload-monitoring -w     # wait until they are Running
```

---

## 2. Create the user directory (`USER_DIR`)

`USER_DIR` is a **separate directory outside the project folder** that holds your customization files (`user.env`, `monitoring/user.env`, `user-values-*.yaml`). SAS's "Create the Deployment Directory" page says it is *not* a subdirectory of the repository, and `samples/README.md` says the same. Keeping it outside means your settings survive when you unpack a newer version of the project.

The project provides a template for it: `samples/generic-base` (every line in it is commented out, so copying it changes nothing by itself). Seed your `USER_DIR` from that template instead of creating the files by hand. Copy only the files this deployment uses:
```bash
cd /home/melsa/viya4-monitoring-kubernetes-1.2.54
export USER_DIR=$HOME/v4m-user
mkdir -p $USER_DIR/monitoring
cp samples/generic-base/user.env                                        $USER_DIR/
cp samples/generic-base/monitoring/user.env                             $USER_DIR/monitoring/
cp samples/generic-base/monitoring/user-values-openshift-grafana.yaml   $USER_DIR/monitoring/
cp samples/generic-base/monitoring/user-values-pushgateway.yaml         $USER_DIR/monitoring/
```
(The rest of `generic-base` also works. It is skipped here because it adds README files inside `monitoring/alerting` and `monitoring/dashboards`, and the scripts treat the contents of those two folders as alert and dashboard files.)

Then edit the copies. Uncomment or add only these lines:

`$USER_DIR/user.env`:
```
LOG_DEBUG_ENABLE=false
# OPENSHIFT_ROUTE_DOMAIN=apps.ocp.lab.datascience.me   # only if step 1.2 looked different
```
`$USER_DIR/monitoring/user.env`:
```
MON_NS=monitoring
# LOGGING_DATASOURCE=false
```
Keep `USER_DIR` exported in every shell you use (or put the `export` line in `~/.bashrc`).

**What happens if `USER_DIR` is not set:** `bin/common.sh` uses `USER_DIR=${USER_DIR:-$(pwd)}`, and the scripts `cd` to the project folder first, so `USER_DIR` silently becomes the **project folder itself**. The scripts then read `<project>/monitoring/user.env` (the shipped, all-commented sample) and never see your `user.env` at all. In Stage B that shows up as `AIRGAP_REGISTRY has not been set`. Check with `echo $USER_DIR` before every run; the script prints `User directory: <path>` near the top of its output, which should be `/home/melsa/v4m-user`.

---

## 3. STAGE A — Connected deployment

```bash
cd /home/melsa/viya4-monitoring-kubernetes-1.2.54
export USER_DIR=$HOME/v4m-user
monitoring/bin/deploy_monitoring_openshift.sh 2>&1 | tee ~/v4m-monitoring-stageA.log
```
For more detail, run once with `LOG_DEBUG_ENABLE=true`.

What you should see, in order:
1. `OpenShift server version: 4.20.x` and the client version
2. `Deploying monitoring to the [monitoring] namespace...`
3. Helm repo add/update for `grafana-community` (needs internet — Stage A only)
4. `Enabling OpenShift user workload monitoring` (or a note that the ConfigMap exists)
5. `grafana-serviceaccount` created, `cluster-monitoring-view` added, token created (`oc create token ... --duration 12000h`)
6. `Deploying Grafana...` (Helm `--atomic`; takes a few minutes for the image pull and the PVC)
7. `Using OpenShift authentication for Grafana`, then `Patching Grafana pod with authenticating TLS proxy...`
8. Dashboards and recording rules applied
9. `Exposing Grafana service as a route...`
10. `Successfully deployed SAS Viya Monitoring for OpenShift` and the Grafana URL

### Verify Stage A

```bash
oc get pods -n monitoring
# v4m-grafana-xxxxx  should become Running with all containers ready
#   (expect grafana, two sidecars, and grafana-proxy — about 4/4)

oc get route -n monitoring v4m-grafana
oc get pvc -n monitoring                       # Bound
oc get cm -n monitoring -l grafana_dashboard   # dashboard ConfigMaps
```
Then in a browser:
1. Open the Route URL (`https://v4m-grafana-monitoring.apps.<domain>`).
2. You should get the **OpenShift login** page, then Grafana.
3. **Connections → Data sources → Prometheus → Save & test** must succeed.
4. **Explore** → run `up`. You should see series.
5. The Viya dashboards are in the dashboards list. They only show data if Viya is running on this cluster (next section).

**Stage A passes when:** Grafana pod is Ready, OpenShift login works, and the `up` query returns data.

### Optional: Viya namespace (only if Viya runs on this cluster)
```bash
export VIYA_NS=<viya-namespace>
monitoring/bin/deploy_monitoring_viya.sh
```
This installs a Pushgateway (with a PVC) and the Viya ServiceMonitors/PodMonitors. Give Prometheus a few minutes, then check the Viya dashboards.

### Stage A troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `Permission denied` on a script | You skipped `chmod +x` (step 0) |
| `bad interpreter: ^M` | CRLF line endings — run `dos2unix` |
| `does not have cluster admin access` | Log in as cluster-admin |
| `Unsupported OpenShift client version` | Use `oc` 4.19 or newer |
| `Required component [yq] not available` / incorrect version | Install mikefarah `yq` 4.45.1+ |
| `Unable to determine OpenShift route host` | Set `OPENSHIFT_ROUTE_DOMAIN` in `$USER_DIR/user.env` |
| Helm `--atomic` timeout, pod `Pending` | PVC not bound — check `oc get pvc,events -n monitoring`. Most often there is no default storage class (see step 1) |
| Grafana pod `CrashLoopBackOff`, log says data dir not writable / `Permission denied` | NFS/CSI volume not writable by OpenShift's random UID. Repeat the storage write test in step 1 and fix the backend permissions |
| Pod `ImagePullBackOff` on `grafana-proxy` | Cluster cannot pull `registry.redhat.io` (check the global pull secret) |
| Prometheus datasource test fails (401/403) | Token or ClusterRoleBinding missing — re-run the script |
| Datasource works but Viya panels are empty | Viya not deployed here, or `deploy_monitoring_viya.sh` not run / user workload monitoring off |

### Messages seen in the lab run that did **not** stop the deployment

| Message / observation | Meaning |
|---|---|
| `Flag --atomic has been deprecated, use --rollback-on-failure instead` | Helm 4 renamed the flag. The project still passes `--atomic`; it is only a warning |
| `Warning: spec.template.spec.containers[3].ports[0]: duplicate port definition with ...containers[0].ports[0]` | Printed by Kubernetes when the oauth-proxy patch is applied to the Grafana deployment. The patch was still applied (`deployment.apps/v4m-grafana patched`). Confirm the pod is healthy with `oc get pods -n monitoring` |
| No `Enabling OpenShift user workload monitoring` line in the output | Expected when you already applied `cluster-monitoring-config` before the run (the script only creates it when it is missing) |
| Browser shows **Not secure** on the Grafana URL | The Route is re-encrypt and uses the cluster's default ingress certificate, which is self-signed in the lab. Import the ingress CA into the browser, or accept the warning |
| The bastion already had Helm repos (`fluent`, `opensearch`, `bitnami`, ...) | `helm repo update` succeeded because they were reachable. In the client's air gap they will not be reachable; see the known risk in B7 |

To start over:
```bash
monitoring/bin/remove_monitoring_openshift.sh
# add MON_DELETE_PVCS_ON_REMOVE=true to also delete the Grafana PVC
# add MON_DELETE_NAMESPACE_ON_REMOVE=true to delete the namespace
```
The namespace (and any pull secret you created in it) is kept by default.

---

## 4. STAGE B — Harbor rehearsal of the air-gap flow

Goal: prove the deployment works when everything comes from a private registry, following the same steps the client will follow with Quay.

Set once (adjust). **Never type the password into this file** (it lives in OneDrive and will be reused for the client runbook); the `read` line prompts for it:
```bash
export REG_HOST=harbor.lab.datascience.me       # host only, no https://, add :port if not 443
export REG=$REG_HOST/monitoring                 # host + the ONE Harbor project that holds everything
export REG_USER='robot$<account-with-push-rights>'   # single quotes: the $ is part of a Harbor robot name
read -rs -p 'Harbor password: ' REG_PASS; export REG_PASS; echo
```
`REG_HOST` is used where a registry **host** is required (login, CA trust, pull secret). `REG` (host + project) is used for image and chart paths.

### B1. Create ONE Harbor project: `monitoring`
Harbor does not auto-create projects, but you only need one (private is fine). Everything goes inside it:

| Path inside project `monitoring` | Holds |
|---|---|
| `grafana/grafana` | Grafana image (and `grafana/tempo` if used) |
| `kiwigrid/k8s-sidecar` | Sidecar image |
| `openshift4/ose-oauth-proxy` | oauth-proxy image |
| `prometheus/pushgateway` | Pushgateway image (Viya option) |
| `grafana-community/grafana` | Grafana Helm chart (and `tempo` if used) |
| `prometheus-community/prometheus-pushgateway` | Pushgateway Helm chart (Viya option) |

**Why this works.** The scripts build every image reference as `AIRGAP_REGISTRY` + `/<original repo>/<image>:<tag>`, and every chart reference as `oci://AIRGAP_HELM_REPO/<chart repo>/<chart>` (`AIRGAP_HELM_REPO` defaults to `AIRGAP_REGISTRY`). The registry value is used as a plain prefix (checked in `bin/common.sh` `generateImageKeysFile`, and in the Grafana, oauth-proxy and Pushgateway templates). So setting `AIRGAP_REGISTRY=<host>/monitoring` (step B6) puts `monitoring/` in front of every path, with no script changes. The repository names become multi-level (`monitoring/grafana/grafana`), which both registries accept, with the conditions below.

**Account rights.** The account you push with needs **push and pull** on project `monitoring`. A robot account whose name says "pull_secret" is normally pull-only, so it cannot push. Use a push-capable account on the bastion, and keep the pull-only robot for the cluster's `v4m-image-pull-secret` (B5).

**Registry support for multi-level names.**
- **Harbor:** push and pull of nested names work (Harbor proxy-cache projects create the same kind of paths). Older Harbor versions had UI/REST quirks with names containing several slashes; they do not affect push/pull.
- **Quay (client):** nested names need the Quay config option `FEATURE_EXTENDED_REPOSITORY_NAMES`, which Red Hat says was added in Quay 3.6 and is set in `config.yaml` by default ([Red Hat Quay release notes](https://docs.redhat.com/en/documentation/red_hat_quay/3.15/html-single/red_hat_quay_release_notes/index)). **Harbor passing does not prove Quay accepts it**, so the client runbook starts with a one-image test push (`<quay-host>/monitoring/test/hello:1`). In Quay the `monitoring` **organization** must exist first; repositories are created on push.

### B2. Trust Harbor's CA (skip if Harbor has a publicly trusted cert)

On the deployment host:
```bash
# OS trust store: covers Podman, skopeo AND Helm (the deploy scripts do not pass --ca-file to Helm)
#   RHEL:    sudo cp harbor-ca.crt /etc/pki/ca-trust/source/anchors/ && sudo update-ca-trust
#   Ubuntu:  sudo cp harbor-ca.crt /usr/local/share/ca-certificates/harbor-ca.crt && sudo update-ca-certificates

# Optional, Podman-only alternative if you do not want to touch the OS trust store
# (the folder name must match $REG_HOST exactly, including :port if there is one):
sudo mkdir -p /etc/containers/certs.d/$REG_HOST && sudo cp harbor-ca.crt /etc/containers/certs.d/$REG_HOST/ca.crt
```
Helm reads only the OS trust store, so do the first step even if you use the Podman-only folder.
In the cluster (so nodes can pull from Harbor; no node reboot):
```bash
oc create configmap registry-cas -n openshift-config --from-file=$REG_HOST=harbor-ca.crt
oc patch image.config.openshift.io/cluster --type=merge -p '{"spec":{"additionalTrustedCA":{"name":"registry-cas"}}}'
```
If Harbor uses a port, the ConfigMap key must replace `:` with `..` (for example `harbor.mylab.local..5000`).

Do **not** use `registrySources.insecureRegistries`; it triggers a node rollout.

### B3. Mirror the images
```bash
echo "$REG_PASS" | podman login $REG_HOST -u "$REG_USER" --password-stdin

# Grafana
podman pull docker.io/grafana/grafana:13.0.3
podman tag  docker.io/grafana/grafana:13.0.3 $REG/grafana/grafana:13.0.3
podman push $REG/grafana/grafana:13.0.3

# Sidecar
podman pull quay.io/kiwigrid/k8s-sidecar:2.10.1
podman tag  quay.io/kiwigrid/k8s-sidecar:2.10.1 $REG/kiwigrid/k8s-sidecar:2.10.1
podman push $REG/kiwigrid/k8s-sidecar:2.10.1
```
(Only for a lab Harbor with a self-signed cert you have **not** trusted in B2: add `--tls-verify=false` to `podman login` and `podman push`.)

`podman pull` fetches the image for the CPU architecture of the host you run it on. That is fine when the bastion and the cluster nodes are both x86_64. Check with `uname -m` and `oc get nodes -o jsonpath='{.items[*].status.nodeInfo.architecture}'`.

**Optional Viya add-on:**
```bash
podman pull quay.io/prometheus/pushgateway:v1.11.3
podman tag  quay.io/prometheus/pushgateway:v1.11.3 $REG/prometheus/pushgateway:v1.11.3
podman push $REG/prometheus/pushgateway:v1.11.3
```

**oauth-proxy** (not handled by `setup_airgap.sh`; needs Red Hat credentials). Reuse the cluster's pull secret for the pull only; the push uses the `podman login` from above:
```bash
oc get secret pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d > redhat-auth.json

podman pull --authfile redhat-auth.json registry.redhat.io/openshift4/ose-oauth-proxy:latest
podman tag  registry.redhat.io/openshift4/ose-oauth-proxy:latest $REG/openshift4/ose-oauth-proxy:latest
podman push $REG/openshift4/ose-oauth-proxy:latest

rm -f redhat-auth.json      # it contains Red Hat credentials
```

If the pull fails with `manifest unknown` (the tag does not exist), list the tags with skopeo (`skopeo list-tags --authfile redhat-auth.json docker://registry.redhat.io/openshift4/ose-oauth-proxy`, before deleting the auth file), mirror a specific one, and put this in `$USER_DIR/user.env`:
```
OPENSHIFT_OAUTHPROXY_FULL_IMAGE="registry.redhat.io/openshift4/ose-oauth-proxy:<tag>"
```
The mirrored path must be `$REG/openshift4/ose-oauth-proxy:<tag>`.

### B4. Mirror the Grafana chart
```bash
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm repo update
mkdir -p ~/v4m-charts && helm pull grafana-community/grafana --version 12.10.4 --destination ~/v4m-charts

echo "$REG_PASS" | helm registry login $REG_HOST -u "$REG_USER" --password-stdin
helm push ~/v4m-charts/grafana-12.10.4.tgz oci://$REG/grafana-community
```
**Optional Viya add-on:**
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts && helm repo update
helm pull prometheus-community/prometheus-pushgateway --version 3.7.0 --destination ~/v4m-charts
helm push ~/v4m-charts/prometheus-pushgateway-3.7.0.tgz oci://$REG/prometheus-community
```
Check it is readable: `helm show chart oci://$REG/grafana-community/grafana --version 12.10.4`.

*Alternative that avoids OCI charts entirely:* set `AIRGAP_HELM_FORMAT=tgz` and `AIRGAP_HELM_REPO=$HOME/v4m-charts` in `user.env`; the script then installs from `$AIRGAP_HELM_REPO/<chart>-<version>.tgz`. Use this if the client's Quay does not accept Helm OCI pushes.

### B5. Remove Stage A and prepare the namespace
```bash
monitoring/bin/remove_monitoring_openshift.sh          # namespace and PVC are kept by default
```
The pull secret must exist **before** the deploy script runs: `airgap-include.sh` checks for it and exits if it is missing, before the script gets to create the namespace.
```bash
oc get ns monitoring || oc create ns monitoring

oc create secret docker-registry v4m-image-pull-secret -n monitoring \
  --docker-server=$REG_HOST --docker-username="$REG_USER" --docker-password="$REG_PASS"
```
Use the registry **host** for `--docker-server` (not `host/monitoring`), and preferably a **pull-only** robot here (see B1, "Account rights"): the cluster only ever needs to pull.
If you also run `deploy_monitoring_viya.sh`, create the same secret in the Viya namespace, because that script checks `VIYA_NS` in air-gap mode.

### B6. Switch on air-gap mode
Append to `$USER_DIR/user.env`:
```
AIRGAP_DEPLOYMENT=true
AIRGAP_REGISTRY=harbor.lab.datascience.me/monitoring
AIRGAP_IMAGE_PULL_SECRET_NAME=v4m-image-pull-secret
AIRGAP_HELM_FORMAT=oci
```
`AIRGAP_REGISTRY` is `<host>/monitoring` (host **plus** the project), the same value as `$REG`. Do not set `AIRGAP_HELM_REPO`; it then defaults to the same value, which matches where B4 pushes the charts.
`AIRGAP_REGISTRY_USERNAME/PASSWORD` are only needed by `setup_airgap.sh`, not by the deploy scripts.

### B7. Deploy from Harbor
```bash
export USER_DIR=$HOME/v4m-user
echo "$REG_PASS" | helm registry login $REG_HOST -u "$REG_USER" --password-stdin   # deploy scripts pull the chart with the cached login
monitoring/bin/deploy_monitoring_openshift.sh 2>&1 | tee ~/v4m-monitoring-stageB.log
```
Expect the log line `Deploying into an 'air-gapped' cluster from private registry [harbor...]`.

**Known risk — `helm repo update`:** `deploy_monitoring_openshift.sh` line 41 (and `deploy_monitoring_viya.sh` line 33) run `helm repo update` even in air-gap mode, under `set -e`. On the host where you added `grafana-community` in B4 this only warns. On a host with no Helm repos configured I expect it to abort with "no repositories found". Fix: run `helm repo add grafana-community https://grafana-community.github.io/helm-charts` once on that host (it does not need to be reachable), or wrap those lines in `if [ "$AIRGAP_DEPLOYMENT" != "true" ]; then ... fi`.

### B8. Verify that everything came from Harbor
```bash
oc get pods -n monitoring -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}'
```
Every image must start with `$REG/`. Repeat the browser checks from Stage A.

To make the rehearsal stricter, block the worker nodes' internet access (or firewall docker.io, quay.io, ghcr.io, registry.redhat.io) and re-run `remove_monitoring_openshift.sh` then the deploy.

**Stage B passes when:** the pod is Ready, every container image is from Harbor, and Grafana works exactly as in Stage A.

### Stage B troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `The image pull secret ... was not detected` | Secret missing in `monitoring` (or the Viya namespace). Create it first (B5) |
| `AIRGAP_REGISTRY has not been set` | Not in `$USER_DIR/user.env`, or `USER_DIR` not exported |
| `ErrImagePull` / `manifest unknown` | Path mismatch — compare the failing image in `oc describe pod` with what you pushed in B3 |
| `x509: certificate signed by unknown authority` from a pod | Cluster does not trust Harbor's CA (B2, `additionalTrustedCA`) |
| `x509` from `helm` on the host | Harbor CA not in the OS trust store |
| `unauthorized` on pull | Wrong pull-secret credentials, or the secret is in the wrong namespace |
| `helm push` / `podman push` denied | Project `monitoring` missing, or the account lacks **push** rights (a "pull_secret" robot is usually pull-only) |
| Push rejected with an "invalid name" style error (Quay) | Nested repository names not enabled: `FEATURE_EXTENDED_REPOSITORY_NAMES` (see B1) |
| `helm repo update` aborts | See the known risk in B7 |

---

## 5. Things to know (also relevant for the client)

- **Token lifetime.** Grafana's access to OpenShift monitoring uses a token created with `--duration 12000h`. OpenShift may cap this (the script's own comment says possibly 12 months). Re-run `deploy_monitoring_openshift.sh` before it expires, or the Prometheus datasource will start failing with 401.
- **Who can log in.** In `monitoring/openshift/grafana-proxy-patch-*.template` the `-openshift-sar` / `-openshift-delegate-urls` lines are commented out, and `grafana-proxy-values.yaml` sets `auto_assign_org_role: Admin`. From reading the files, that means any user who can authenticate to OpenShift gets Grafana **Admin**. Please confirm this on your test cluster (log in with a non-admin OpenShift user) and decide whether the client needs it tightened.
- **Static cookie secret.** `grafana-proxy-secret.yaml` contains a fixed `session_secret` that is published in the repo. Replace it if the client's security team objects.
- **`latest` tags.** oauth-proxy and busybox (logging only) use `:latest`. Pin the digest or tag you tested when you carry images to the client.

## 6. Internal lab log (what we found, kept for the client runbook)

Environment: OpenShift 4.20, bastion host `bastion` (Linux, user `melsa`), cluster domain `apps.ocp.lab.datascience.me`.

Versions used in the Stage A run (all accepted by the scripts): OpenShift server 4.20.37, `oc` client 4.19.0 (the minimum allowed for a 4.20 server), Kubernetes server v1.33.13, `kubectl` v1.32.1, Helm 4.1.3. The log line `User directory: /home/melsa/v4m-user` confirmed `USER_DIR` was picked up.

Lab paths (used in the commands above):

| Item | Path on the bastion |
|---|---|
| Project (repo root — run every script from here) | `/home/melsa/viya4-monitoring-kubernetes-1.2.54` |
| `USER_DIR` (outside the repo; seeded from `samples/generic-base`) | `/home/melsa/v4m-user` (`$HOME/v4m-user`) |
| Chart `.tgz` folder (Stage B) | `/home/melsa/v4m-charts` |
| Deploy logs | `/home/melsa/v4m-monitoring-stageA.log`, `/home/melsa/v4m-monitoring-stageB.log` |

The scripts `cd` to the repo root themselves, but `chmod` (section 0) and `oc apply -f monitoring/openshift/...` need you to be in it first. The client runbook will use the client's own paths.

| Date | Check | Result | Consequence |
|---|---|---|---|
| 2026-09-20 | `oc get sc` | Only `truenas-nfs` (`csi.truenas.io`, `Retain`, `Immediate`, expansion allowed); **no default** | Must set a default class (or per-app storage class) before deploying |
| 2026-09-20 | Console route host | `console-openshift-console.apps.ocp.lab.datascience.me` | Script derives `apps.ocp.lab.datascience.me` correctly; no `OPENSHIFT_ROUTE_DOMAIN` needed. Grafana URL will be `https://v4m-grafana-monitoring.apps.ocp.lab.datascience.me` |
| 2026-09-20 | `cluster-monitoring-config` in `openshift-monitoring` | Not found | Script (or the manual `oc apply` above) creates it with `enableUserWorkload: true` |
| 2026-09-20 | Storage write test on `truenas-nfs` | Not run as a separate test. Indirect evidence: Grafana started on its PVC and an OpenShift user was auto-created in Grafana (that writes to the database on the PVC) | Confirm with `oc get pvc,pods -n monitoring` (PVC `Bound`, pod not restarting) |
| 2026-09-20 11:16 | **Stage A (connected) — deploy script** | **Completed:** `Successfully deployed SAS Viya Monitoring for OpenShift`. Helm releases `v4m-grafana` and `v4m-metrics` installed (revision 1). Route `https://v4m-grafana-monitoring.apps.ocp.lab.datascience.me` reachable, OpenShift login worked, 13 dashboards listed (OpenSearch, PostgreSQL, RabbitMQ, SAS CAS/Go/Java/Arke/Launched Jobs/Micro Analytic/Viya Welcome) | Deployment works |
| 2026-09-20 | Registry layout decision | Harbor host `harbor.lab.datascience.me`; **one project `monitoring`** for all images and charts, via `AIRGAP_REGISTRY=harbor.lab.datascience.me/monitoring` (Podman used for mirroring, on the lab and at the client) | Client needs a Quay organization `monitoring` and nested repository names enabled; verify with a test push first |
| — | Stage A — remaining checks | *pending:* pod containers all Ready, Prometheus datasource "Save & test", `up` query in Explore returns data | Stage A is fully passed only after these three |
| — | Stage B (Harbor) | *pending* | |

Browser note: the machine you browse from must resolve `*.apps.ocp.lab.datascience.me` (DNS or hosts file), or the OpenShift login page will not load.

## 7. When you are done

Send me the result of both stages (pass/fail, and the log file or the failing message if any). I will then write the client-side runbook, including the export/import steps for moving images and charts into the client's Quay, with any corrections from what actually happened.
