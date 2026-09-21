# Runbook: SAS Viya Monitoring (Grafana) on an Air-Gapped OpenShift 4.20 Cluster

Project: SAS Viya Monitoring for Kubernetes **1.2.54**. Scope: **metric monitoring** on OpenShift (Grafana with OpenShift login, reading OpenShift's built-in monitoring). Logging (OpenSearch) is not covered.

Registry: the client's **Red Hat Quay**. Container tool: **Podman**.

---

## 0. Read this first

### 0.1 What gets installed

OpenShift already runs Prometheus, so this project does **not** install one. It installs into one namespace (default `monitoring`):

| Component | Purpose |
|---|---|
| Grafana (Helm chart `grafana` 12.10.4, image `grafana` 13.0.3) with a 5Gi PVC | Dashboards |
| `k8s-sidecar` (2 containers in the Grafana pod) | Loads dashboards and datasources from ConfigMaps |
| `ose-oauth-proxy` (container in the Grafana pod) | OpenShift login for Grafana. **Optional:** part 10, Option 3 turns it off and uses Grafana's own users; the image is then not needed |
| Service account `grafana-serviceaccount` + `cluster-monitoring-view` | Lets Grafana query OpenShift's Thanos Querier |
| SAS Viya dashboards, one `PrometheusRule`, a `v4m-grafana` Route | |

Optional (part 9): a Prometheus Pushgateway and the Viya ServiceMonitors/PodMonitors in the Viya namespace.

OpenShift's **user workload monitoring** must be enabled (part 2.4).

### 0.2 What was tested, and what was not

This procedure was run end to end on an OpenShift 4.20.37 lab cluster, first connected, then with every image and the Helm chart coming from a private Harbor registry (`AIRGAP_DEPLOYMENT=true`, one project `monitoring`, Podman, Helm 4.1.3). Both deployments completed. In the second one, the chart and all four pod images came from the registry.

These points were **not** proven in the lab. Each has a check in this runbook that you run **before** the deployment, so none of them should surprise you:

| Not proven in the lab | Where it is checked here |
|---|---|
| Quay accepting multi-level repository names (`monitoring/grafana/grafana`) | Part 3.2 (test push) |
| A registry on a **non-standard port** (`:8443`; the lab Harbor used 443): login, CA trust, pull secret, the cluster's CA ConfigMap key | Parts 3.2, 3.3, 5.2 and 8.2 |
| Grafana with OpenShift login turned off (Option 3 in part 10) | Part 10 (run it once on staging and record the result) |
| Quay accepting Helm charts as OCI artifacts | Part 4.3 (fallback: `tgz` mode) |
| `helm repo update` (run by the deploy script) behaving on a host with no internet | Part 6 |
| A cluster with **no** internet access at all (the lab cluster had it) | Part 8.2 |
| Moving images and charts across the air gap as files | Parts 1 and 4 |
| Viya metrics appearing in the dashboards | Part 9 |

### 0.3 Access and decisions you need

- [ ] **Cluster-admin** on the OpenShift cluster (the scripts refuse to run without it).
- [ ] A Linux **bastion** inside the air gap that can reach the OpenShift API and Quay.
- [ ] A **connected** machine (Linux, x86_64 preferred) to download the artifacts, and an approved way to carry files across (checksums are in part 1.5).
- [ ] A **Red Hat login** on the connected machine (for `registry.redhat.io`, part 1.2).
- [ ] **Quay**: permission to create an organization named `monitoring`, and two accounts: one that can push, one that can only pull (part 3.1).
- [ ] The client's approval to enable OpenShift **user workload monitoring** if it is not already on.
- [ ] A default **storage class** (or the name of the class to use) for one 5Gi volume.
- [ ] The client's decision on **who may use Grafana** (part 10). By default every OpenShift user who can log in becomes Grafana Admin. Options include turning OpenShift login off and using only local Grafana users (Option 3).

### 0.4 Conventions

- Commands run in `bash`. `<angle-brackets>` are values you replace.
- **Never write the registry password into a file or a shell history you keep.** The commands below prompt for it.
- Paths: `/opt/v4m` is used as the working folder on both machines. Change it if you prefer.
- `<quay-host>` in this document means the value of `REG_HOST` in 0.5 (host **and port**).

### 0.5 Values for this client

| Item | Value |
|---|---|
| Quay URL | `https://stg-sacv4-quay.staging.dubaipolice.com:8443` |
| `REG_HOST` (host **and port**, no `https://`) | `stg-sacv4-quay.staging.dubaipolice.com:8443` |
| `REG` (host + organization) | `stg-sacv4-quay.staging.dubaipolice.com:8443/monitoring` |
| `AIRGAP_REGISTRY` in `user.env` | same as `REG` |
| `--docker-server` in the pull secret | `stg-sacv4-quay.staging.dubaipolice.com:8443` |
| Key in the cluster's CA ConfigMap (part 3.3) | `stg-sacv4-quay.staging.dubaipolice.com..8443` (the colon is written as two dots) |
| Example image path | `stg-sacv4-quay.staging.dubaipolice.com:8443/monitoring/grafana/grafana:13.0.3` |

Because the port is 8443, **every** reference to the registry must include `:8443` (login, pull secret, `AIRGAP_REGISTRY`), and only the cluster CA ConfigMap key uses `..` instead of `:`. The lab registry used the standard port, so these spots were not exercised there; parts 3.2, 3.3 and 8.2 check them.

The name (`stg-...`, `staging`) suggests this is the **staging** Quay. If the production cluster uses a different Quay, change `REG_HOST` and repeat from part 3.

---

## 1. Connected machine: build the transfer bundle

### 1.1 Tools

`podman`, `helm` (3.8 or newer; 4.1.3 was tested), `sha256sum`, `tar`.

```bash
mkdir -p /opt/v4m/bundle && cd /opt/v4m/bundle
podman version && helm version --short
```

### 1.2 Images

Create the list. The first three are required; the fourth only for part 9:

```bash
cat > images.txt <<'EOF'
docker.io/grafana/grafana:13.0.3
quay.io/kiwigrid/k8s-sidecar:2.10.1
registry.redhat.io/openshift4/ose-oauth-proxy:latest
quay.io/prometheus/pushgateway:v1.11.3
EOF
```
(Remove the `pushgateway` line if you will not do part 9. If you will use Option 3 in part 10, OpenShift login off, you also do not need the `ose-oauth-proxy` line or the `podman login registry.redhat.io` below.)

Log in to Red Hat's registry (required for the oauth-proxy image), pull, and save:
```bash
podman login registry.redhat.io          # Red Hat account or registry service account

while read -r img; do podman pull --arch amd64 "$img"; done < images.txt

podman save --multi-image-archive -o v4m-monitoring-images.tar $(cat images.txt)
ls -lh v4m-monitoring-images.tar
```
`--arch amd64` makes sure you get x86_64 images even if the connected machine is not. Check the cluster's architecture later (part 2.4); if the nodes are not x86_64, change it.

Record exactly what you carried (the oauth-proxy tag is `latest`, which moves over time):
```bash
while read -r img; do echo "$img  $(podman image inspect --format '{{.Digest}}' "$img")"; done < images.txt | tee images-digests.txt
```

### 1.3 Helm charts

```bash
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm repo update
mkdir -p charts
helm pull grafana-community/grafana --version 12.10.4 --destination charts

# Only for part 9 (Pushgateway):
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm pull prometheus-community/prometheus-pushgateway --version 3.7.0 --destination charts

ls -l charts
```

### 1.4 The project and the tools for the bastion

- The project folder `viya4-monitoring-kubernetes-1.2.54`. **Create the archive on Linux**, from the folder that contains the project folder, so that the executable bit on the scripts is kept (a zip made on Windows loses it). Put the archive inside the bundle folder:
  ```bash
  chmod +x viya4-monitoring-kubernetes-1.2.54/bin/*.sh viya4-monitoring-kubernetes-1.2.54/monitoring/bin/*.sh viya4-monitoring-kubernetes-1.2.54/logging/bin/*.sh
  tar czf /opt/v4m/bundle/v4m-project-1.2.54.tar.gz viya4-monitoring-kubernetes-1.2.54
  ```
- `helm` and `yq` (**mikefarah**, version **4.45.1 or newer**) binaries for the bastion, if it does not have them, also saved into `/opt/v4m/bundle`. Get `yq_linux_amd64` from https://github.com/mikefarah/yq/releases and `helm-v<version>-linux-amd64.tar.gz` from https://get.helm.sh.
- `oc` is not needed here: the bastion can download it from the cluster's console (part 2.1).

### 1.5 Checksums

```bash
cd /opt/v4m && find bundle -type f ! -name SHA256SUMS -exec sha256sum {} + > bundle/SHA256SUMS
```
Carry the whole `bundle` folder across the air gap.

---

## 2. Bastion: prepare

### 2.1 Tools

Required on the bastion:

| Tool | Version | Notes |
|---|---|---|
| `bash`, `sha256sum`, `openssl` | any | |
| `oc` | **4.19 or newer** | The cluster serves it: OpenShift console → **?** → **Command line tools**. This works without internet |
| `kubectl` | 1.27+ | Comes in the same archive as `oc` |
| `helm` | 3.8+ | 4.1.3 tested |
| `yq` (mikefarah) | 4.45.1+ | |
| `podman` | any recent | |

```bash
oc version; kubectl version --client; helm version --short; yq --version; podman --version
```

### 2.2 Unpack and verify

```bash
mkdir -p /opt/v4m && cp -r <transfer-media>/bundle /opt/v4m/ && cd /opt/v4m/bundle
sha256sum -c SHA256SUMS                 # every line must say OK

tar xzf v4m-project-1.2.54.tar.gz -C /opt/v4m
cd /opt/v4m/viya4-monitoring-kubernetes-1.2.54

# scripts must be executable and use Unix line endings (this must print nothing):
chmod +x bin/*.sh monitoring/bin/*.sh logging/bin/*.sh
grep -rIl $'\r' --include=*.sh . | head
```
If files are listed, run `dos2unix` on them.

### 2.3 Log in as cluster-admin

```bash
oc login https://api.<cluster-domain>:6443 -u <admin-user>
oc whoami
oc auth can-i create namespace --all-namespaces      # must print: yes
```

### 2.4 Cluster checks

```bash
# a) Versions. Server must be 4.14+; oc client must be 4.19+ for a 4.20 server.
oc version

# b) Node CPU architecture: must match what you pulled in part 1.2 (amd64)
oc get nodes -o jsonpath='{.items[*].status.nodeInfo.architecture}{"\n"}'

# c) Storage: one class must show "(default)"
oc get sc

# d) Console route: the script derives the Grafana route domain from it
oc get route -n openshift-console console -o jsonpath='{.spec.host}{"\n"}'
#    expected: console-openshift-console.apps.<domain>

# e) User workload monitoring
oc -n openshift-monitoring get cm cluster-monitoring-config -o yaml
```

**c) No default storage class?** Either mark one as default:
```bash
oc patch storageclass <name> -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```
or set the class only for Grafana (part 5.3, `storageClassName`). Without one, the Grafana PVC stays `Pending` and the install times out.

**c) Storage write test (recommended).** A bound volume does not prove a pod running under OpenShift's restricted SCC (random UID) can write to it. NFS-backed classes are the usual problem. Use any image already available in the client's registry (replace `<image>`, it must contain `sh` and `touch`):
```bash
oc create ns pvc-test
cat <<EOF | oc apply -f -
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
    image: <image>
    command: ["sh","-c","touch /data/x && echo WRITE_OK; sleep 3600"]
    volumeMounts: [{name: d, mountPath: /data}]
  volumes: [{name: d, persistentVolumeClaim: {claimName: t}}]
EOF
oc get pvc,pod -n pvc-test ; oc logs -n pvc-test t      # expect: WRITE_OK
oc delete ns pvc-test                                    # then delete a leftover PV if the class uses reclaim policy Retain
```

**e) User workload monitoring.**
- `NotFound`: nothing to do now; enable it in part 2.5.
- The ConfigMap **exists**: the script will **not** change it. Open it and make sure `config.yaml` contains `enableUserWorkload: true`, **keeping every other setting**:
  ```bash
  oc -n openshift-monitoring edit cm cluster-monitoring-config
  ```
  **Do not `oc apply` the project's `cluster-monitoring-config.yaml` over an existing ConfigMap**; it would replace the client's other monitoring settings.

### 2.5 Enable user workload monitoring (only if the ConfigMap did not exist)

With the client's approval:
```bash
cd /opt/v4m/viya4-monitoring-kubernetes-1.2.54
oc apply -f monitoring/openshift/cluster-monitoring-config.yaml
oc get pods -n openshift-user-workload-monitoring -w        # wait until they are Running (Ctrl+C to stop)
```

---

## 3. Quay: prepare

### 3.1 Organization and accounts

In Quay:
(Menu names differ slightly between Quay versions; the concepts are the same.)

1. **Organization.** Web UI: **+** (top right) → **New Organization** → name `monitoring` (plus an e-mail address). Everything (images and charts) lives in it. Repositories are created on the first push.
2. **Push account** (used only on the bastion). It must be allowed to *create* repositories in `monitoring`, and Quay's team role for that is **Creator** (Red Hat's roles are Member, Creator and Admin: [Quay permissions model](https://docs.redhat.com/en/documentation/red_hat_quay/3.13/html/managing_access_and_permissions/role-based-access-control)):
   - Organization `monitoring` → **Teams and Membership** → create a team `pushers` with role **Creator**.
   - Add the person (or a robot account) that will push to that team.
   - Or simply use an organization **owner** for the mirroring, then stop using it.
3. **Pull-only robot** (goes into the cluster, part 5.2). Robots are limited to one organization and get *None*, *Read*, *Write* or *Admin* per repository ([Quay robot accounts](https://docs.redhat.com/en/documentation/red_hat_quay/3/html/managing_access_and_permissions/allow-robot-access-user-repo)).
   - Organization → **Robot Accounts** → **Create Robot Account** → name `pull`. Its user name is `monitoring+pull`; copy its token from the robot's credentials view.
   - The six repositories do not exist yet, so give the robot *Read* on them automatically: organization → **Default Permissions** → create a rule "repositories created by *the push account* (or *Anyone*)", grant **Read** to `monitoring+pull`.
   - If you skip the default permission, add `monitoring+pull` with **Read** to each repository after part 4 (Repository → **Settings** → permissions).
   - Check at the end (8.2): the pod must pull its images with this robot; `unauthorized` means the Read permission is missing.
4. Ask the Quay administrator to confirm **`FEATURE_EXTENDED_REPOSITORY_NAMES`** is enabled in `config.yaml`. Red Hat documents nested repository names from Quay 3.6, on by default ([release notes](https://docs.redhat.com/en/documentation/red_hat_quay/3.15/html-single/red_hat_quay_release_notes/index)). Without it the image paths used here are rejected.

The repository paths used (all inside organization `monitoring`):

| Path | Content |
|---|---|
| `grafana/grafana` | Grafana image |
| `kiwigrid/k8s-sidecar` | sidecar image |
| `openshift4/ose-oauth-proxy` | oauth-proxy image |
| `prometheus/pushgateway` | Pushgateway image (part 9) |
| `grafana-community/grafana` | Grafana Helm chart |
| `prometheus-community/prometheus-pushgateway` | Pushgateway Helm chart (part 9) |

If pushes are denied because repositories cannot be created on push, create these six repositories in advance (Quay UI: organization `monitoring` → **Create New Repository**; use the exact names above, including the slash, for example `grafana/grafana`), and give the push account *Write* and `monitoring+pull` *Read* on each. Repositories created by a push are **private** by default, which is what you want: the cluster authenticates with the pull secret.

### 3.2 Shell variables and the test push

```bash
export REG_HOST=stg-sacv4-quay.staging.dubaipolice.com:8443   # host:port, no https://
export REG=$REG_HOST/monitoring                   # host + organization
export REG_USER='<push-account>'                  # single quotes: robot names contain a $
read -rs -p 'Quay password/token: ' REG_PASS; export REG_PASS; echo
```
Keep these exported in every shell you use for parts 3 to 8.

**Get Quay's CA certificate.** Ask the client's PKI or Quay team for the PEM file of the CA that signed Quay's certificate (called `quay-ca.crt` below). If you cannot get it, you can read what the server presents; check with the client's security team that it is the right one before trusting it:
```bash
openssl s_client -connect $REG_HOST -servername ${REG_HOST%%:*} -showcerts </dev/null 2>/dev/null \
  | sed -n '/BEGIN CERTIFICATE/,/END CERTIFICATE/p' > quay-chain.pem
openssl s_client -connect $REG_HOST -servername ${REG_HOST%%:*} </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates
```
`quay-chain.pem` holds the server certificate and any intermediates. Trusting the issuing CA is better than trusting the server certificate, which changes when it is renewed.

Trust Quay's CA on the bastion (skip if Quay has a certificate from a CA the OS already trusts). This covers Podman **and** Helm:
```bash
#   RHEL:    sudo cp quay-ca.crt /etc/pki/ca-trust/source/anchors/ && sudo update-ca-trust
#   Ubuntu:  sudo cp quay-ca.crt /usr/local/share/ca-certificates/quay-ca.crt && sudo update-ca-certificates
```

**Test push** (proves login, CA trust, permission to create repositories, and multi-level names). Use any image already on the bastion (`podman images`). If there is none, run the `podman load` command from part 4.1 first and use the sidecar image:
```bash
echo "$REG_PASS" | podman login $REG_HOST -u "$REG_USER" --password-stdin
podman tag <any-local-image> $REG/test/hello:1
podman push $REG/test/hello:1
```
- `Login Succeeded` and the push completes: go on. Delete `test/hello` in Quay afterwards.
- Push rejected with an invalid-name error: nested names are off; see 3.1 step 4.
- `x509` error: the CA is not trusted on the bastion.
- `denied` / `unauthorized`: the account lacks push or create rights on `monitoring`.

### 3.3 Make the cluster trust Quay's CA

Needed only if Quay's certificate is not signed by a CA the cluster already trusts. This does **not** reboot nodes.
```bash
oc create configmap registry-cas -n openshift-config --from-file=${REG_HOST/:/..}=quay-ca.crt
oc patch image.config.openshift.io/cluster --type=merge -p '{"spec":{"additionalTrustedCA":{"name":"registry-cas"}}}'
```
- If a ConfigMap `registry-cas` (or any `additionalTrustedCA`) already exists, **add your key to it** instead of creating another. Check first: `oc get image.config.openshift.io/cluster -o jsonpath='{.spec.additionalTrustedCA}'`.
- Quay uses port 8443, so the ConfigMap key must be `stg-sacv4-quay.staging.dubaipolice.com..8443` (a ConfigMap key cannot contain a colon). `${REG_HOST/:/..}` in the command above does that replacement for you. Check the result: `oc get cm registry-cas -n openshift-config -o jsonpath='{.data}' | head -c 120`, the key must contain `..8443`.
- Do **not** use `registrySources.insecureRegistries`; it triggers a node rollout.

---

## 4. Load the images and charts into Quay

### 4.1 Images

```bash
cd /opt/v4m/bundle
podman load -i v4m-monitoring-images.tar

echo "$REG_PASS" | podman login $REG_HOST -u "$REG_USER" --password-stdin

while read -r img; do
  target="$REG/${img#*/}"          # drop the source registry host, keep <repo>/<image>:<tag>
  podman tag  "$img" "$target"
  podman push "$target"
done < images.txt
```
Example: `docker.io/grafana/grafana:13.0.3` becomes `<quay-host>/monitoring/grafana/grafana:13.0.3`. This is the exact path the scripts will look for.

Self-signed Quay you did **not** trust in 3.2? Add `--tls-verify=false` to `podman login` and `podman push` for this step only.

### 4.2 Check the images

```bash
while read -r img; do t="$REG/${img#*/}"; podman rmi "$t" >/dev/null; podman pull "$t" >/dev/null && echo "OK  $t"; done < images.txt
```
Every line must say `OK`.

### 4.3 Helm charts

```bash
echo "$REG_PASS" | helm registry login $REG_HOST -u "$REG_USER" --password-stdin

helm push charts/grafana-12.10.4.tgz oci://$REG/grafana-community
# Part 9 only:
helm push charts/prometheus-pushgateway-3.7.0.tgz oci://$REG/prometheus-community

# Check it can be read back:
helm show chart oci://$REG/grafana-community/grafana --version 12.10.4
```
**If Quay refuses the chart push** (OCI chart support is disabled or restricted on that Quay): do not spend time on it, use the local-file mode instead. Copy `charts/*.tgz` to `/opt/v4m/charts` on the bastion and set these two lines in `user.env` (part 5.3) instead of the OCI default:
```
AIRGAP_HELM_FORMAT=tgz
AIRGAP_HELM_REPO=/opt/v4m/charts
```
The scripts then install from `/opt/v4m/charts/grafana-12.10.4.tgz` and do not need the charts in Quay.

---

## 5. Namespace, pull secret and settings

### 5.1 Namespace
```bash
oc get ns monitoring || oc create ns monitoring
```
(Use another name if you prefer, and set `MON_NS` in 5.3 accordingly.)

### 5.2 Image pull secret

The deploy script **stops** if this secret is missing, so it must exist before you deploy. Use the registry **host** for `--docker-server` (not `host/monitoring`) and a **pull-only** account:
```bash
oc create secret docker-registry v4m-image-pull-secret -n monitoring \
  --docker-server=$REG_HOST --docker-username='<pull-only-robot>' --docker-password='<its-token>'
```
(Avoid leaving the token in shell history: prefix the command with a space if `HISTCONTROL=ignorespace` is set, or use `read -rs` into a variable as in 3.2.)

### 5.3 The settings folder (`USER_DIR`)

`USER_DIR` is a folder **outside** the project that holds your settings. If you do not set it, the scripts silently use the project folder itself and will not see your `user.env`.

```bash
cd /opt/v4m/viya4-monitoring-kubernetes-1.2.54
export USER_DIR=/opt/v4m/user-dir
mkdir -p $USER_DIR/monitoring
cp samples/generic-base/user.env                                      $USER_DIR/
cp samples/generic-base/monitoring/user.env                           $USER_DIR/monitoring/
cp samples/generic-base/monitoring/user-values-openshift-grafana.yaml $USER_DIR/monitoring/
cp samples/generic-base/monitoring/user-values-pushgateway.yaml       $USER_DIR/monitoring/
```
Append to `$USER_DIR/user.env` (use `vi`, and use `=` not `:`; the sample file has a typo, `AIRGAP_HELM_FORMAT: oci`, which the shell rejects):
```
AIRGAP_DEPLOYMENT=true
AIRGAP_REGISTRY=stg-sacv4-quay.staging.dubaipolice.com:8443/monitoring
AIRGAP_IMAGE_PULL_SECRET_NAME=v4m-image-pull-secret
AIRGAP_HELM_FORMAT=oci
```
- `AIRGAP_REGISTRY` is the host **plus** `/monitoring`, identical to `$REG`.
- Do not set `AIRGAP_HELM_REPO` for OCI mode; it defaults to `AIRGAP_REGISTRY`.
- Optional in `$USER_DIR/user.env`: `OPENSHIFT_ROUTE_DOMAIN=apps.<domain>` only if the console route in 2.4 d) did not look like `console-openshift-console.apps.<domain>`.

Append to `$USER_DIR/monitoring/user.env`:
```
MON_NS=monitoring
# Only if you chose Option 3 in part 10 (no OpenShift login, local Grafana users only):
# OPENSHIFT_AUTH_ENABLE=false
```

**No default storage class and you chose the per-Grafana setting:** edit `$USER_DIR/monitoring/user-values-openshift-grafana.yaml`, uncomment `persistence:` and set:
```yaml
persistence:
  storageClassName: <your-storage-class>
```

**If oauth-proxy was mirrored under a specific tag** (not `latest`), also add to `$USER_DIR/user.env`:
```
OPENSHIFT_OAUTHPROXY_FULL_IMAGE="registry.redhat.io/openshift4/ose-oauth-proxy:<tag>"
```
The image in Quay must then be `<quay-host>/monitoring/openshift4/ose-oauth-proxy:<tag>`.

---

## 6. Pre-flight: the deploy script's `helm repo update`

`monitoring/bin/deploy_monitoring_openshift.sh` (line 41) and `deploy_monitoring_viya.sh` (line 33) run `helm repo update` even in air-gap mode, with `set -e`. The lab bastion could reach the internet, so this was **not** exercised. Test it on the client's bastion; the commands use throw-away Helm config files and change nothing:

```bash
# 1. No repositories configured
HELM_REPOSITORY_CONFIG=$(mktemp) HELM_REPOSITORY_CACHE=$(mktemp -d) helm repo update; echo "exit code: $?"

# 2. A repository configured but unreachable
cat > /tmp/dead-repos.yaml <<'EOF'
apiVersion: ""
generated: "2026-01-01T00:00:00Z"
repositories:
- name: dead
  url: http://127.0.0.1:9
EOF
HELM_REPOSITORY_CONFIG=/tmp/dead-repos.yaml HELM_REPOSITORY_CACHE=$(mktemp -d) helm repo update; echo "exit code: $?"
```
- Both tests print `exit code: 0`: nothing to do.
- **Either** test prints a non-zero exit code: make the update conditional (it is a small, safe change). In `monitoring/bin/deploy_monitoring_openshift.sh` (and `deploy_monitoring_viya.sh` if you use part 9), change
  ```bash
  helm repo update
  ```
  to
  ```bash
  if [ "$AIRGAP_DEPLOYMENT" != "true" ]; then helm repo update; fi
  ```
  Record the change; it is lost when the project is upgraded.

(You can also see it fail in the real run in part 7 and come back here.)

---

## 7. Deploy

```bash
cd /opt/v4m/viya4-monitoring-kubernetes-1.2.54
export USER_DIR=/opt/v4m/user-dir
echo "$REG_PASS" | helm registry login $REG_HOST -u "$REG_USER" --password-stdin      # skip in tgz mode

monitoring/bin/deploy_monitoring_openshift.sh 2>&1 | tee ~/v4m-monitoring-deploy.log
```
The push account is fine for `helm registry login`; the Helm chart is pulled by this shell, not by the cluster.

**Check these lines in the output** (they show the settings were picked up):
- `INFO User directory: /opt/v4m/user-dir`. If it shows the project folder, `USER_DIR` was not exported.
- `Deploying into an 'air-gapped' cluster from private registry [<quay-host>/monitoring]`
- `Pulled: <quay-host>/monitoring/grafana-community/grafana:12.10.4` (OCI mode)
- No `export: ... not a valid identifier` line. If you see one, a line in `user.env` uses `:` instead of `=`.
- Last line: `Successfully deployed SAS Viya Monitoring for OpenShift`, followed by the Grafana URL.

Harmless messages seen in the lab:
- `Flag --atomic has been deprecated, use --rollback-on-failure instead` (Helm 4)
- `Warning: ... duplicate port definition ...` when the oauth-proxy patch is applied

---

## 8. Verify

### 8.1 Pods and volume

```bash
oc get pods,pvc -n monitoring
```
The Grafana pod is `Running` with all its containers ready (4/4: `grafana`, two sidecars, `grafana-proxy`), few or no restarts, and the PVC is `Bound`.

### 8.2 Everything comes from Quay

```bash
oc get pods -n monitoring -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}'
```
Every image must begin with `<quay-host>/monitoring/`. This is the proof that nothing was taken from the internet. In a cluster that truly has no internet, a wrong path shows up as `ImagePullBackOff` instead.

### 8.3 Grafana

```bash
oc get route -n monitoring v4m-grafana
```
1. Open the URL (`https://v4m-grafana-monitoring.apps.<domain>`). Your workstation must resolve `*.apps.<domain>`. You get the **OpenShift login**, then Grafana. A browser warning is normal if the ingress certificate is signed by an internal CA the browser does not trust.
2. **Connections → Data sources → Prometheus → Save & test** must succeed.
3. **Explore**, run `up`. It must return series.
4. **Dashboards** lists the SAS Viya dashboards (CAS, Go, Java, Arke, Launched Jobs, Postgres, RabbitMQ, OpenSearch, Welcome). They stay empty until Viya metrics exist (part 9).

**The deployment is complete when 8.1, 8.2, 8.3 (steps 1 to 3) pass.**

If you deployed **Option 3** (OpenShift login off): the pod has 3 containers (no `grafana-proxy`), the image list in 8.2 has no `openshift4/ose-oauth-proxy`, and step 1 shows the Grafana **login form** instead of the OpenShift login. Log in as `admin` (part 10, Option 3, steps 6 and 7).

### 8.4 Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `Permission denied` on a script | Executable bit missing (2.2) |
| `bad interpreter: ^M` | Windows line endings: `dos2unix` |
| `The image pull secret ... was not detected` | Secret missing in `monitoring` (5.2) |
| `AIRGAP_REGISTRY has not been set` | `USER_DIR` not exported, or the line is missing in `$USER_DIR/user.env` |
| `does not have cluster admin access` | Log in as cluster-admin |
| `Unsupported OpenShift client version` | `oc` older than 4.19 |
| `Required component [yq] ...` | Install mikefarah `yq` 4.45.1+ |
| Script stops at `helm repo update` | Part 6 |
| Helm `x509` error | Quay CA not in the bastion OS trust store (3.2) |
| Helm cannot find the chart | Chart not pushed to `oci://$REG/grafana-community`, or use tgz mode (4.3) |
| Pod `ImagePullBackOff`, `manifest unknown` | Image missing at `<quay-host>/monitoring/<repo>/<image>:<tag>`; compare `oc describe pod` with part 4.1 |
| Pod `ImagePullBackOff`, `x509` | Cluster does not trust Quay's CA (3.3) |
| Pod `ImagePullBackOff`, `unauthorized` | Wrong pull-secret credentials, the robot has no *Read* on `monitoring`, or the secret's `--docker-server` is missing the port (`...com:8443`) |
| Helm `--atomic` timeout, PVC `Pending` | No default storage class (2.4 c) |
| Grafana pod crash-looping, `Permission denied` in its log | The storage does not allow OpenShift's random UID to write; repeat the write test in 2.4 c) and fix the storage backend |
| Login page never loads | DNS for `*.apps.<domain>` from your workstation |
| Prometheus datasource test fails (401/403) | Token or ClusterRoleBinding missing: re-run the deploy script |
| Data source works but `up` is empty | User workload monitoring not running (2.4 e / 2.5) |
| Option 3: the Route shows "Application is not available", or the pod stays in `ContainerCreating` | The TLS secret is missing. `oc get svc,secret -n monitoring \| grep v4m-grafana`: `v4m-grafana-tls-secret` must exist (OpenShift creates it from the Service annotation). `oc describe pod -n monitoring -l app.kubernetes.io/name=grafana` shows a `FailedMount` if it does not |
| Option 3: `admin` login is refused | Read the password: `oc get secret -n monitoring v4m-grafana -o jsonpath='{.data.admin-password}' \| base64 -d; echo`. If you changed it in Grafana afterwards, the Secret still holds the old value |

---

## 9. Optional: Viya namespace (Pushgateway and monitors)

Needed for the Viya dashboards to show data. Requires the Pushgateway image (1.2) and chart (1.3), both already loaded in part 4.

```bash
export VIYA_NS=<viya-namespace>

# the pull secret must also exist in the Viya namespace (the script checks there in air-gap mode)
oc create secret docker-registry v4m-image-pull-secret -n $VIYA_NS \
  --docker-server=$REG_HOST --docker-username='<pull-only-robot>' --docker-password='<its-token>'

cd /opt/v4m/viya4-monitoring-kubernetes-1.2.54
monitoring/bin/deploy_monitoring_viya.sh 2>&1 | tee ~/v4m-monitoring-viya.log
```
This creates a Pushgateway with a volume in the Viya namespace, and the ServiceMonitors/PodMonitors for the Viya pods.

Verify:
```bash
oc get pods -n $VIYA_NS | grep -i pushgateway
oc get servicemonitor,podmonitor -n $VIYA_NS
```
Then wait a few minutes and check the SAS dashboards in Grafana. If they stay empty, check that user workload monitoring is running (`oc get pods -n openshift-user-workload-monitoring`) and that Viya's network policies allow the user workload Prometheus to reach the Viya pods' metrics ports.

---

## 10. Security: decide before users get access

**Finding (lab, 2026-09-20):** with the default configuration, **any user who can log in to OpenShift can open Grafana, and is created with the Grafana Admin role** (they see Administration, Users, Teams, Service accounts). The oauth-proxy is deployed without an authorization check, and the project's Grafana setting `auto_assign_org_role` is `Admin`. Two related points, the proxy cookie secret and Grafana's token lifetime, are in 10.2 and 10.3.

Choose one of the three options below. Options 1 and 2 have not been tested at all. Option 3 uses the project's own `OPENSHIFT_AUTH_ENABLE=false` setting, whose script path was reviewed but has not been run in the lab yet. Test whichever you choose with a non-admin user before relying on it.

**Option 1: only some OpenShift users may log in.** The proxy template has commented examples for exactly this. In `monitoring/openshift/grafana-proxy-patch-host.template` (use `grafana-proxy-patch-path.template` if `OPENSHIFT_PATH_ROUTES=true`), add one line under `args:`, below the commented examples:
```yaml
        - '-openshift-sar={"namespace":"monitoring","resource":"services","resourceName":"v4m-grafana","verb":"get"}'
```
Then re-run `monitoring/bin/deploy_monitoring_openshift.sh` (the script re-applies the template each run, so a change made only with `oc patch` would be lost), and grant that access to the intended group only:
```bash
oc -n monitoring create role grafana-users --verb=get --resource=services --resource-name=v4m-grafana
oc -n monitoring create rolebinding grafana-users --role=grafana-users --group=<openshift-group>
```
Test: a user in the group can log in; a user outside it is refused. The exact key names in the JSON depend on the oauth-proxy version; if the pod's proxy container rejects the argument, check its log (`oc logs -n monitoring deploy/v4m-grafana -c grafana-proxy`).

**Option 2: everyone who logs in gets a lower Grafana role.** In `$USER_DIR/monitoring/user-values-openshift-grafana.yaml`:
```yaml
"grafana.ini":
  users:
    auto_assign_org_role: Viewer
```
Effects: nobody can edit dashboards or datasources (the built-in `admin` login form is disabled by the OpenShift proxy configuration), and the setting only applies to users created **after** it is set, so apply it before anyone has logged in. Use it only if read-only is acceptable, or combined with Option 1.

**Option 3: turn OpenShift login off and use only local Grafana users.** This is the project's own switch, `OPENSHIFT_AUTH_ENABLE=false`, described in SAS's "Red Hat OpenShift Considerations" page. Nobody logs in with OpenShift credentials; every person is a Grafana user created by a Grafana administrator. It also removes the "any OpenShift user is Admin" problem, because there is no automatic sign-up.

| | OpenShift login (default) | Option 3 |
|---|---|---|
| Sign-in | OpenShift OAuth through `ose-oauth-proxy` | Grafana login form (user name and password) |
| Containers in the Grafana pod | 4 (grafana, two sidecars, `grafana-proxy`) | 3 (no proxy) |
| `ose-oauth-proxy` image | required, from `registry.redhat.io` | **not needed**: leave it out of 1.2 and 4.1 |
| HTTPS | the proxy terminates it | Grafana serves HTTPS itself on port 3001 with a certificate OpenShift issues for the Service; the Route re-encrypts to it |
| Users | created on first login, as Admin | created by an administrator; sign-up is off; new users get Grafana's default role (Viewer) |
| Admin | none | the built-in `admin`, with a password you choose or Grafana generates |

Steps:

1. **Start clean if OpenShift login was already deployed.** The proxy container was patched into the Deployment, and the users that OpenShift logins created (as Admin) are stored on Grafana's volume. Dashboards are recreated from ConfigMaps, so nothing of value is lost:
   ```bash
   MON_DELETE_PVCS_ON_REMOVE=true monitoring/bin/remove_monitoring_openshift.sh
   ```
   Skip this on a first deployment.
2. **Set the switch** in `$USER_DIR/monitoring/user.env` (part 5.3): `OPENSHIFT_AUTH_ENABLE=false`
3. **Choose the admin password.** Either let Grafana generate one (nothing to do; step 6 shows how to read it), or set it for this deployment only, without writing it in a file:
   ```bash
   read -rs -p 'Grafana admin password: ' GRAFANA_ADMIN_PASSWORD; export GRAFANA_ADMIN_PASSWORD; echo
   ```
   The script hands it to Helm with `--set adminPassword`, so it is also stored in the Helm release Secret in `monitoring`. Limit who can read secrets in that namespace, and change the password after the first login (step 9).
4. **Deploy** (part 7, same command). What differs in the output: `Creating the Grafana service; annotations will trigger generation of TLS certs.` and `Using native Grafana authentication`; there is no `Patching Grafana pod with authenticating TLS proxy`. If you did not set a password, the script prints how to read the generated one.
5. **Check:**
   ```bash
   oc get pods,svc,route -n monitoring                      # Grafana pod 3/3
   oc get secret -n monitoring v4m-grafana-tls-secret       # created by OpenShift from the Service annotation; must exist
   ```
6. **Log in** at the Route URL as user `admin`. If Grafana generated the password:
   ```bash
   oc get secret -n monitoring v4m-grafana -o jsonpath='{.data.admin-password}' | base64 -d; echo
   ```
7. **Create the users.** In Grafana: **Administration → Users and access → Users → New user** (name, e-mail, user name, initial password), then set the role on the user's page if it must be higher than Viewer. Or by API (`-k` only because the Route uses an internal CA):
   ```bash
   curl -sk -u "admin:$GRAFANA_ADMIN_PASSWORD" -H 'Content-Type: application/json' \
     -X POST https://v4m-grafana-monitoring.apps.<domain>/api/admin/users \
     -d '{"name":"Jane Doe","email":"jane@example.com","login":"jane","password":"<initial-password>"}'
   ```
   Ask each person to change the password at first login (profile → Change password).
8. **Verify** as in 8.3: run the datasource test and the `up` query as `admin`, then log in as one of the new users and open a dashboard.
9. **Change the admin password later** in the Grafana UI, or:
   ```bash
   oc exec -n monitoring deploy/v4m-grafana -c grafana -- grafana cli admin reset-admin-password '<new-password>'
   ```
   The Secret `v4m-grafana` then still holds the old value. The project script `monitoring/bin/change_grafana_admin_password.sh` also updates the Secret, but it was written for the non-OpenShift deployment (it restarts pods selected by a Prometheus-operator label), so on OpenShift it may fail at the restart step after it has already changed the password. It has not been tried here.

What you accept with Option 3: users and passwords are managed by hand in Grafana (no link to the client's identity provider, no automatic removal of people who leave); the login page is reachable by everyone who can reach the Route, so use strong passwords and, if needed, restrict network access to the Route; Grafana's token to OpenShift's Prometheus (10.3) still applies. To go back to OpenShift login, remove `OPENSHIFT_AUTH_ENABLE=false`, remove the deployment with `MON_DELETE_PVCS_ON_REMOVE=true`, and deploy again (the `ose-oauth-proxy` image must then be in Quay).

### 10.2 Cookie secret
`monitoring/openshift/grafana-proxy-secret.yaml` contains a fixed `session_secret` that is published in the project. If the client's security team objects, replace the value with a new base64-encoded random string before deploying, and record the change. Not used in Option 3 (there is no proxy).

### 10.3 Grafana's token
Grafana reads OpenShift's metrics with a service-account token created with a 12000-hour lifetime. OpenShift may cap it (the script's own comment says possibly to 12 months). Put a reminder in the client's calendar to **re-run `deploy_monitoring_openshift.sh` before it expires**; if it does expire, the Prometheus datasource fails with 401.

---

## 11. Operations

**Upgrade.** Unpack the new project version, mirror its new images and charts (parts 1 to 4, using its `component_versions.env`), keep the same `USER_DIR`, re-run the deploy script. Re-apply any script edit from parts 6 and 10.

**Remove.**
```bash
cd /opt/v4m/viya4-monitoring-kubernetes-1.2.54 && export USER_DIR=/opt/v4m/user-dir
monitoring/bin/remove_monitoring_openshift.sh
# MON_DELETE_PVCS_ON_REMOVE=true       also deletes the Grafana volume
# MON_DELETE_NAMESPACE_ON_REMOVE=true  also deletes the namespace (and the pull secret)
```
The cluster-role binding `grafana-serviceaccount-monitoring-binding` and the `cluster-monitoring-view` binding are left behind; delete them by hand if you want a clean cluster.

**`latest` tag.** The oauth-proxy image is `:latest`. Keep `images-digests.txt` from part 1.2 with the deployment records.

---

## 12. Sign-off checklist

- [ ] Bundle checksums verified (2.2)
- [ ] Cluster-admin login, `oc` 4.19+, `yq` 4.45.1+, Helm 3.8+ (2.1, 2.3)
- [ ] Default storage class (or per-Grafana class) and write test passed (2.4)
- [ ] User workload monitoring running (2.4 e / 2.5)
- [ ] Quay organization `monitoring`, test push passed (3.1, 3.2)
- [ ] Images pushed and pulled back (4.2); chart pushed and readable, or tgz mode chosen (4.3)
- [ ] Pull secret created (5.2); `user.env` written with `=` not `:` (5.3)
- [ ] `helm repo update` behaviour checked (6)
- [ ] Deployment completed (7); pods Ready (8.1); all images from Quay (8.2)
- [ ] Login, datasource test and `up` query pass (8.3)
- [ ] Viya monitors deployed and dashboards show data (9), if in scope
- [ ] Access decision recorded and applied (10)
- [ ] Token-renewal reminder set (10.3); digests file stored (11)
