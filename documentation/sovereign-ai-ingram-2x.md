# Deploying the Sovereign AI platform on the Ingram Micro cluster (RHOAI 2.x)

This procedure deploys the platform layer only, GitOps and Red Hat OpenShift AI
2.x, on an Ingram Micro hosted cluster, and verifies GPU support. It does not
deploy any AI application. Deploy the research application once the platform is
verified healthy: see [Next steps](#next-steps).

The procedure uses GitOps (ArgoCD) throughout. Ingram provides the worker nodes
and owns NFD and the GPU Operator, so GPU support is verified here, not
installed.

Stages 1–3 ran successfully on this cluster on 2026-11-06.

## Prerequisites

- The `oc`, `kustomize` and `yq` CLIs installed locally.

- The `kubeadmin` password for the cluster, from Ingram. Do not commit it.

- A login to the cluster. Use the API URL **without a port**, which connects on
  443:

  ```bash
  oc login --server=<api-url> -u kubeadmin
  ```

  > **Note.** The console's "Copy login command" includes a port that may be
  > closed from outside. Remove the port from the command.

- Cluster admin access. Test with:

  ```bash
  oc whoami
  ```

  Expected output: `kube:admin`.

- This repository cloned locally, on the `ingram-2x` branch:

  ```bash
  git clone https://github.com/DevoteamNL/sovereign-ai-gitops.git
  cd sovereign-ai-gitops
  git checkout ingram-2x
  ```

## Procedure overview

| Stage | Description | Wall time | Active work |
|---|---|---|---|
| 1 | Verify the environment | 5 min | 5 min |
| 2 | Bootstrap GitOps | 5 min | 2 min |
| 3 | Wait for the platform stack | 5 min | passive |
| **Total** | | **~15 min** | **~7 min** |

---

## Stage 1: Verify the environment

1. Confirm the OpenShift version:

   ```bash
   oc get clusterversion
   ```

   Expected: `4.18.x`, `Available: True`.

2. Confirm the nodes and their GPUs:

   ```bash
   oc get nodes -L hypershift.openshift.io/nodePool,nvidia.com/gpu.count,nvidia.com/gpu.product
   ```

   Expected: every node `Ready`, one with `GPU.COUNT` of 1 and
   `NVIDIA-H200-141C`, and one with `GPU.COUNT` of 2 and `NVIDIA-H200-35C`. The
   research procedure finds the node pools by these GPU names, so update them
   there if they differ.

3. Confirm the GPU Operator is healthy before you start:

   ```bash
   oc get clusterpolicy gpu-cluster-policy -o jsonpath='{.status.state}{"\n"}'
   ```

   Expected: `ready`. If not, contact Ingram: the GPU Operator is theirs.

4. Confirm the default StorageClass:

   ```bash
   oc get sc
   ```

   Expected: `hcp-sc (default)`.

## Stage 2: Bootstrap GitOps

1. From the `sovereign-ai-gitops` repository, on branch `ingram-2x`, run:

   ```bash
   BOOTSTRAP_DIR=sovereign-ai-ingram-2x ./bootstrap.sh
   ```

   Expected output ends with:

   ```
   GitOps has successfully deployed!  Check the status of the sync here:
   https://openshift-gitops-server-openshift-gitops.<cluster-domain>
   ```

   > **Note.** The script compares your branch with `targetRevision` in the
   > overlay, which is `ingram-2x`. If it offers to update the branch, you are
   > on the wrong one: answer **No** and check out `ingram-2x`.

2. Get the ArgoCD URL and the `admin` password:

   ```bash
   echo "ArgoCD UI: https://$(oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}')"
   oc extract secret/openshift-gitops-cluster -n openshift-gitops --keys=admin.password --to=-
   ```

## Stage 3: Wait for the platform stack

ArgoCD now reconciles Red Hat OpenShift AI 2.x and its own configuration.

1. Watch the Applications converge to `Synced` and `Healthy`:

   ```bash
   watch -n 5 'oc get applications.argoproj.io -n openshift-gitops'
   ```

   Expected: `cluster-config-app-of-apps`, `openshift-ai-operator`,
   `openshift-gitops-operator` and `user-workload-monitoring`. This took about
   5 minutes on 2026-11-06.

   > **Note.** `openshift-ai-operator` can show `Unknown` or `Degraded` for the
   > first few minutes, while the OpenShift AI components start. It recovers on
   > its own.

   > **Important.** There must be no `nfd-operator` or `nvidia-gpu-operator`.
   > Those belong to Ingram, and deploying them from this repository would
   > overwrite their vGPU driver configuration.

2. Verify the GPU nodes are schedulable:

   ```bash
   oc get nodes -l nvidia.com/gpu.present=true \
     -o custom-columns=NAME:.metadata.name,SCHEDULABLE:.spec.unschedulable
   ```

   Expected: both nodes with `SCHEDULABLE: <none>`. If `true`, see the **GPU node
   is cordoned** entry in Troubleshooting.

3. Verify the GPU driver pods are Running:

   ```bash
   oc get pods -n nvidia-gpu-operator -l app.kubernetes.io/component=nvidia-driver -o wide
   ```

   Expected: one pod per node, `2/2` and `Running`.

4. Verify the driver is loaded:

   ```bash
   for pod in $(oc get pods -n nvidia-gpu-operator -l app.kubernetes.io/component=nvidia-driver -o name); do
     oc exec -n nvidia-gpu-operator "$pod" -c nvidia-driver-ctr -- nvidia-smi -L
   done
   ```

   Expected: three GPUs in total, one `NVIDIA H200-141C` and two
   `NVIDIA H200-35C`.

5. Verify OpenShift AI is on 2.25 and Ready:

   ```bash
   oc get csv -n redhat-ods-operator | grep rhods
   oc get datasciencecluster -A
   ```

   Expected: `rhods-operator.2.25.x` `Succeeded`, and `READY: True`.

6. Confirm the GPU Operator was not touched and nothing is stuck:

   ```bash
   oc get clusterpolicy gpu-cluster-policy -o jsonpath='{.status.state}{"\n"}'
   oc get pods -A --field-selector status.phase=Pending
   ```

   Expected: `ready`, and no Pending pods.

The platform is now deployed and verified. No AI workload is running yet.

## Troubleshooting

**`oc login` times out.**
The port in the console's login command may be closed from outside. Use the URL
without a port, as shown in Prerequisites.

**A command stops with `dial tcp <api-ip>:443: i/o timeout`.**
Seen a few times, with `oc get`, `oc exec` and `oc apply`, while other commands
kept working. Run the command again. `bootstrap.sh` is safe to rerun.

**GPU node is cordoned (`SchedulingDisabled`).**
Check whether the driver upgrade failed, then contact Ingram before changing
anything, because the GPU Operator is theirs:

```bash
oc get node <gpu-node-name> -o jsonpath='{.metadata.labels.nvidia\.com/gpu-driver-upgrade-state}{"\n"}'
```

**Login fails with "x509: certificate signed by unknown authority".**
Add `--insecure-skip-tls-verify=true` to the `oc login` command.

## Cluster lifecycle reminders

- **Ingram may restart or rebuild the VMs** to enlarge the disk. Agree the timing
  with them. Workloads should come back on their own, but model images are pulled
  again on a rebuilt node.
- **How long the environment stays available** is not known yet. Ask Ingram and
  record it here.

## Next steps

- **Deploy the research application**: follow `docs/user-docs/deploy-ingram.md`
  in the `sovereign-ai-research` repository, branch `sovereign-option-a`.
