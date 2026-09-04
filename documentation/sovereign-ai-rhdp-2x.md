# Deploying the Sovereign AI platform on a fresh RHDP cluster (RHOAI 2.x)

This procedure deploys the platform layer only — GitOps, GPU support, and
Red Hat OpenShift AI 2.x — on a freshly-provisioned Red Hat Demo Platform
(RHDP) cluster. It does not deploy any AI application. Add a tenant
(`tenants/<name>/`) and its entry in `patch-configs-list.yaml` once the
platform is verified healthy.

The procedure uses GitOps (ArgoCD) throughout. Only the GPU MachineSet is
applied imperatively, since it requires the per-cluster `${CLUSTER_ID}` and
`${AMI_ID}` to be substituted at apply time.

## Prerequisites

- The `oc`, `kustomize`, and `yq` CLIs installed locally. `oc` and
  `kustomize` are pre-installed on RHDP. Install `yq` on Fedora with
  `sudo dnf install yq -y`.

- A freshly-provisioned RHDP cluster ("OpenShift Container Platform with
  IBM Fusion" or equivalent) in `us-east-2`, running OpenShift **4.18.x**.
  RHOAI 2.x does not require 4.19+; if your cluster is already on 4.19+,
  use the `sovereign-ai-rhdp-3x` overlay instead (branch
  `rhoai-3x-archive` — parked until RHDP offers a 4.19+-capable catalog
  item as of this writing).

- Cluster admin access. Test with:

  ```bash
  oc whoami
  ```

  Expected output: `kube:admin`, `kubeadmin`, or `admin` depending on how
  RHDP provisioned the cluster.

- This repository cloned locally, on the `rhoai-2x` branch:

  ```bash
  git clone https://github.com/DevoteamNL/sovereign-ai-gitops.git
  cd sovereign-ai-gitops
  git checkout rhoai-2x
  ```

  No GitHub credentials are required beyond a plain clone — this repo is
  public, so ArgoCD needs no repository credential Secret to read it.

## Procedure overview

| Stage | Description | Wall time | Active work |
|---|---|---|---|
| 1 | Capture cluster context | 2 min | 2 min |
| 2 | Provision GPU nodes | 15 min | 2 min |
| 3 | Bootstrap GitOps | 5 min | 2 min |
| 4 | Wait for platform stack | 10–15 min | passive |
| **Total** | | **~35–40 min** | **~6 min** |

Stages 2 and 3–4 run in parallel — GPU nodes provision in the background
while the platform stack converges.

---

## Stage 1: Capture cluster context

1. Confirm authentication and cluster identity:

   ```bash
   oc whoami
   oc cluster-info | head -1
   ```

2. Capture the cluster infrastructure ID, worker AMI ID, and AZ into shell
   variables. All three are required in Stage 2:

   ```bash
   export CLUSTER_ID=$(oc get infrastructure cluster -o jsonpath='{.status.infrastructureName}')
   echo "Cluster ID: ${CLUSTER_ID}"

   export AMI_ID=$(oc get machineset -n openshift-machine-api \
     -o jsonpath='{.items[0].spec.template.spec.providerSpec.value.ami.id}')
   echo "AMI ID: ${AMI_ID}"

   export AZ=$(oc get machineset -n openshift-machine-api \
     -o jsonpath='{.items[0].spec.template.spec.providerSpec.value.placement.availabilityZone}')
   echo "AZ: ${AZ}"
   ```

   > **Note.** Keep this terminal session open through the rest of the
   > procedure. If you open a new terminal, re-run all three export
   > commands.

3. Verify worker nodes are present and Ready:

   ```bash
   oc get nodes -l node-role.kubernetes.io/worker
   ```

   Expected: at least 3 worker nodes with `STATUS: Ready`.

## Stage 2: Provision GPU nodes

Apply the MachineSet for GPU-enabled workers (`g6e.2xlarge`, NVIDIA L40S,
48GB VRAM each).

> **Confirmed on a real run**: RHDP's base worker nodes do not have
> headroom for ArgoCD's platform pods — `openshift-gitops-application-controller`
> and others land `Pending` with `Insufficient cpu`/`Insufficient memory`
> until the GPU taint is temporarily removed in step 4 below. Treat step 4
> as a required part of this stage, not an optional fallback — don't skip
> it expecting to add a plain worker node instead; using the GPU nodes'
> already-provisioned spare capacity is free, whereas an extra worker node
> is not.

1. Substitute the cluster ID into the MachineSet template and apply:

   ```bash
   envsubst < clusters/overlays/sovereign-ai-rhdp-2x/machinesets/g6e-machineset.yaml \
     | oc apply -f -
   ```

2. Watch the MachineSet provision the nodes (background, 10–15 minutes):

   ```bash
   oc get machinesets -n openshift-machine-api -w
   ```

   Wait until the new MachineSet shows `AVAILABLE` and `READY` both equal
   to `2`. Press `Ctrl+C` to exit.

3. Verify the GPU nodes are Ready and tainted:

   ```bash
   oc get nodes -l node-role.kubernetes.io/gpu='' \
     -o custom-columns=NAME:.metadata.name,STATUS:.status.conditions[-1].type,TAINTS:.spec.taints[*].key
   ```

4. Remove the GPU taint (required — see note above). Patch the
   MachineSet and existing Machine objects first — the nodelink controller
   syncs
   `Machine.spec.taints` → `Node.spec.taints` on every reconcile cycle, so
   patching only the MachineSet is not enough:

   ```bash
   oc patch machineset ${CLUSTER_ID}-g6e-${AZ} -n openshift-machine-api \
     --type json -p '[{"op":"remove","path":"/spec/template/spec/taints"}]'

   for m in $(oc get machines -n openshift-machine-api \
     -l machine.openshift.io/cluster-api-machineset=${CLUSTER_ID}-g6e-${AZ} \
     -o name); do
     oc patch $m -n openshift-machine-api \
       --type merge -p '{"spec":{"taints":null}}'
   done

   oc adm taint nodes -l node-role.kubernetes.io/gpu='' nvidia.com/gpu-
   ```

   Restore the taint in Stage 4 once the platform stack is healthy.

## Stage 3: Bootstrap GitOps

1. Run the bootstrap script. This installs the OpenShift GitOps operator,
   configures ArgoCD, and applies the cluster Applications:

   ```bash
   BOOTSTRAP_DIR=sovereign-ai-rhdp-2x ./bootstrap.sh
   ```

   Expected output ends with:

   ```
   GitOps has successfully deployed!  Check the status of the sync here:
   https://openshift-gitops-server-openshift-gitops.<cluster-domain>
   ```

   This takes ~3–5 minutes.

   > **Important — branch check.** The script compares your current git
   > branch against the `targetRevision` baked into
   > `clusters/overlays/sovereign-ai-rhdp-2x/kustomization.yaml` (currently
   > `main`). If you're running this from the `rhoai-2x` branch (or any
   > branch other than `main`), it will detect the mismatch and prompt:
   > ```
   > Your current working branch is rhoai-2x, and your cluster overlay
   > branch is main.
   > Do you wish to update it to rhoai-2x?
   > ```
   > Answering **Yes** edits `kustomization.yaml`, commits, and **pushes
   > to `origin` automatically** — this is what makes ArgoCD track your
   > actual branch instead of `main`. Answer Yes when working from a
   > feature branch (`yq` prerequisite for this check — see below).

   > **Note.** `bootstrap.sh` needs `yq`, or it fails partway through.
   > No `yq`? Skip the script and run the steps below by hand instead.
   > First confirm
   > `targetRevision` in the overlay's `kustomization.yaml` matches the
   > branch you're deploying from (the script would otherwise do this
   > check for you), then:
   >
   > 1. Install the GitOps operator, if not already installed:
   >    ```bash
   >    kustomize build components/operators/openshift-gitops/operator/overlays/latest/ | oc apply -f -
   >    ```
   > 2. Apply the cluster overlay and wait for ArgoCD:
   >    ```bash
   >    kustomize build bootstrap/overlays/sovereign-ai-rhdp-2x/ | oc apply -f -
   >    oc wait --for=condition=Available deployment/openshift-gitops-server \
   >      -n openshift-gitops --timeout=300s
   >    oc delete pods -l app.kubernetes.io/name=openshift-gitops-application-controller \
   >      -n openshift-gitops
   >    oc wait --for=condition=Available deployment/openshift-gitops-server \
   >      -n openshift-gitops --timeout=300s
   >    ```

2. Get the ArgoCD UI URL for monitoring progress:

   ```bash
   echo "ArgoCD UI: https://$(oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}')"
   ```

   Log in with `kubeadmin` (password from RHDP).

## Stage 4: Wait for the platform stack

ArgoCD now reconciles Node Feature Discovery (NFD), the NVIDIA GPU
Operator, and Red Hat OpenShift AI (RHOAI) 2.x.

1. Watch Applications converge to `Synced` and `Healthy`:

   ```bash
   watch -n 5 'oc get applications.argoproj.io -n openshift-gitops'
   ```

   > **Note.** Use `applications.argoproj.io`, not `applications` — on
   > RHDP the bare `applications` resource can resolve to the IBM
   > Spectrum Fusion CRD and return no results.

   This typically takes 10–15 minutes.

   > **Known issue — `nvidia-gpu-operator` shows `OutOfSync/Missing`:**
   > ArgoCD applies the `ClusterPolicy` CR before the `nvidia.com/v1` CRD
   > is available (the GPU Operator CSV takes ~6 minutes to reach
   > `Succeeded`). ArgoCD exhausts its retries before then. Once the CSV
   > is `Succeeded`, trigger a manual sync:
   > ```bash
   > oc get csv -n nvidia-gpu-operator | grep gpu-operator
   > oc patch application.argoproj.io nvidia-gpu-operator -n openshift-gitops \
   >   --type merge -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"syncStrategy":{"hook":{}}}}}'
   > ```

2. Verify NFD applied GPU labels and nodes are schedulable (if Stage 2 was
   run):

   ```bash
   oc get nodes -l nvidia.com/gpu.present=true \
     -o custom-columns=NAME:.metadata.name,SCHEDULABLE:.spec.unschedulable
   ```

   Expected: `SCHEDULABLE: <none>`. If `true`, see the **GPU node is
   cordoned** entry in Troubleshooting.

3. Verify GPU Operator driver pods are Running:

   ```bash
   oc get pods -n nvidia-gpu-operator -l app.kubernetes.io/component=nvidia-driver -o wide
   ```

4. Verify the driver is loaded:

   ```bash
   GPU_POD=$(oc get pods -n nvidia-gpu-operator -l app.kubernetes.io/component=nvidia-driver -o name | head -1)
   oc exec -n nvidia-gpu-operator ${GPU_POD} -c nvidia-driver-ctr -- nvidia-smi
   ```

   Expected: an `nvidia-smi` table showing NVIDIA L40S, 48GB VRAM.

5. Verify RHOAI is Ready:

   ```bash
   oc get datasciencecluster -A
   ```

   Expected: `READY: True` (columns are `NAME`/`READY`/`REASON`, not
   `Phase`).

6. Restore the GPU taint now that the platform stack is healthy
   (required — mandatory counterpart to Stage 2 step 4; skipping it
   leaves the GPU nodes open to non-GPU workloads indefinitely):

   ```bash
   export CLUSTER_ID=$(oc get infrastructure cluster -o jsonpath='{.status.infrastructureName}')
   export AZ=$(oc get machineset -n openshift-machine-api \
     -o jsonpath='{.items[0].spec.template.spec.providerSpec.value.placement.availabilityZone}')

   oc patch machineset ${CLUSTER_ID}-g6e-${AZ} -n openshift-machine-api \
     --type json -p '[{"op":"add","path":"/spec/template/spec/taints","value":[{"key":"nvidia.com/gpu","value":"true","effect":"NoSchedule"}]}]'

   oc adm taint nodes -l node-role.kubernetes.io/gpu='' nvidia.com/gpu=true:NoSchedule
   ```

The platform is now deployed and verified. No AI workload is running yet.

## Troubleshooting

**GPU node is cordoned (`SchedulingDisabled`).**
The GPU Operator's upgrade-controller can cordon a node and park it there
even when the driver is actually loaded and working — commonly triggered
by an RHDP auto-stop/auto-resume cycle. Symptom: node shows
`Ready,SchedulingDisabled` and carries
`nvidia.com/gpu-driver-upgrade-state=upgrade-failed`.

```bash
NODE=<gpu-node-name>
oc get node $NODE -o jsonpath='{.metadata.labels.nvidia\.com/gpu-driver-upgrade-state}{"\n"}'
# Confirm GPU operator components on the node are Running, then:
oc label node $NODE nvidia.com/gpu-driver-upgrade-state=upgrade-done --overwrite
oc adm uncordon $NODE
```

**GPU nodes are Ready but `nvidia-smi` fails inside a pod.**
The GPU Operator's driver DaemonSet hasn't placed pods on the new nodes
yet. Wait 5 more minutes. If pods still aren't there:

```bash
oc get pods -n nvidia-gpu-operator -o wide
oc describe node <gpu-node-name> | grep -i taint
```

The GPU Operator tolerates `nvidia.com/gpu=true:NoSchedule`. If the taint
differs, the DaemonSet won't schedule.

**MachineSet creates Machines but they never become Nodes.**
RHDP's IAM is scoped to Machine API operations only.

```bash
oc get machines -n openshift-machine-api
oc describe machine <name> -n openshift-machine-api
```

Common cause: AWS capacity or quota for the requested instance type. Try
a different instance type if `g6e.2xlarge` is unavailable in your AZ.

**Login fails with "x509: certificate signed by unknown authority".**
Add `--insecure-skip-tls-verify=true` to the `oc login` command. RHDP
clusters often use self-signed certs.

## Cluster lifecycle reminders

- **Auto-stop is 6 hours** from provisioning. The cluster pauses; resume
  from the RHDP order page.
- **Auto-destroy is 30 hours.** Provision a fresh one and re-run this
  procedure from Stage 1.
- **Cluster ID is per-provision.** Re-run Stage 1, step 2 every time you
  log into a new cluster.
- **GPU node bills run while the cluster is up**, even when idle. Stop it
  from RHDP when not actively using it.

## Next steps

- **Add an AI workload tenant**: create `tenants/<name>/` (an ArgoCD
  `Application` pointing at the app repo) and add its entry to
  `components/argocd/apps/overlays/sovereign-ai-rhdp-2x/patch-configs-list.yaml`.
  ArgoCD picks it up automatically on the next sync.
