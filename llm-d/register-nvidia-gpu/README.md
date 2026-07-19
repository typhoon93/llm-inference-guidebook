# Kubernetes GPU Quick Check
Reference: https://github.com/NVIDIA/k8s-device-plugin

## Step 1 --- Does Linux see the NVIDIA GPU?

Run:

``` bash
nvidia-smi
```

**Expected:** GPU information is displayed.

-   ✅ Success → Linux sees the GPU.
-   ❌ Failure → Fix the NVIDIA driver before troubleshooting
    Kubernetes.

------------------------------------------------------------------------

## Step 2 --- Does Kubernetes see the GPU?

Run:

``` bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu
```

**Expected:**

``` text
NAME           GPU
<node-name>    1
```

or another positive number.

-   ✅ A number is shown → Kubernetes can schedule GPU workloads.
-   ❌ `<none>` or an empty value → Kubernetes does not see the GPU.

# K8s Register NVIDIA GPU

## 1. Verify the host can see the GPU

```bash
nvidia-smi
```

Expected result: the NVIDIA GPU, driver version, and CUDA version are shown.

## 2. Install the NVIDIA Container Toolkit

```bash
sudo dnf install -y nvidia-container-toolkit
```

Verify:

```bash
nvidia-ctk --version
rpm -qa | grep -Ei 'nvidia-container|nvidia-container-toolkit'
```

## 3. Install `runc`

The NVIDIA container runtime needs an OCI runtime available in the normal system `PATH`.

```bash
sudo dnf install -y runc
```

Verify:

```bash
which runc
runc --version
nvidia-container-runtime --version
```

## 4. Restart k3s

```bash
sudo systemctl restart k3s
sudo systemctl status k3s --no-pager
```

## 5. Confirm the NVIDIA RuntimeClass exists

```bash
kubectl get runtimeclass
```

Expected entry:

```text
NAME      HANDLER
nvidia    nvidia
```


## 6. Install the NVIDIA Device Plugin

Reference: https://github.com/NVIDIA/k8s-device-plugin#deployment-via-helm

Add and update the official NVIDIA Helm repository:

```bash
helm repo add nvdp https://nvidia.github.io/k8s-device-plugin
helm repo update
```

Install the NVIDIA Device Plugin with GPU Feature Discovery enabled:

```bash
helm upgrade --install nvdp \
  nvdp/nvidia-device-plugin \
  --namespace nvidia-device-plugin \
  --create-namespace \
  --version 0.17.1 \
  --set runtimeClassName=nvidia \
  --set gfd.enabled=true
```

This installs:

- The NVIDIA Device Plugin
- Node Feature Discovery
- GPU Feature Discovery
- Automatic NVIDIA GPU node labels

## 7. Verify the installation

Check that the NVIDIA components are running:

```bash
kubectl get pods -n nvidia-device-plugin
```

Expected components include:

```text
node-feature-discovery-master
node-feature-discovery-worker
nvidia-device-plugin
gpu-feature-discovery
```

Check that Kubernetes advertises the GPU:

```bash
kubectl describe node rhel9-hpomen \
  | grep -A10 -E 'Capacity:|Allocatable:'
```

Expected:

```text
Capacity:
  nvidia.com/gpu: 1

Allocatable:
  nvidia.com/gpu: 1
```

## 8. Test GPU access from a pod

Use the `gpu-test.yaml` from this folder (cd into it);

Run the test:

```bash
kubectl apply -f gpu-test.yaml
kubectl wait --for=condition=Ready pod/gpu-test --timeout=120s
kubectl logs gpu-test
```

Expected result: `nvidia-smi` displays the GPU from inside the pod after the last command

Clean up:

```bash
kubectl delete -f gpu-test.yaml
```

## Notes

- Kubernetes allocates whole GPU devices through `nvidia.com/gpu`; it does not schedule GPU memory independently.
- This node advertises one GPU, so a pod requesting `nvidia.com/gpu: 2` will remain `Pending`.
- Headlamp may omit extended resources such as `nvidia.com/gpu` from its summary view. `kubectl describe node` shows the authoritative Kubernetes API value.
- The MPS control DaemonSet may show zero desired pods when NVIDIA MPS is not enabled. This is expected.