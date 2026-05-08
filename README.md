# Mutating Admission Policy and Agent Sandbox

This guide demonstrates how to use [CEL MutatingAdmissionPolicy](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionpolicy) to enforce the shape of an [agent sandbox](https://github.com/kubernetes-sigs/agent-sandbox) pod template. It identifies sandboxes with a specific runtime class and rewrites the pod template to use a specific image, something that could be useful with runtime classes that want to ensure compatible images are admitted.

## Prerequisites

- [Kind](https://kind.sigs.k8s.io/) installed.
- [kubectl](https://kubernetes.io/docs/tasks/tools/) installed.

## Step 1: Create a Kind Cluster

Our demo is using kind which supports MutatingAdmissionPolicy, but with some cluster config at the moment:

```bash
cat <<EOF | kind create cluster --name sandbox-cluster --config -
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
featureGates:
  MutatingAdmissionPolicy: true
kubeadmConfigPatches:
- |
  kind: ClusterConfiguration
  apiServer:
    extraArgs:
      runtime-config: "admissionregistration.k8s.io/v1beta1=true"
EOF
```


## Step 2: Install Agent Sandbox

Follow the instructions from the [agent-sandbox repository](https://github.com/kubernetes-sigs/agent-sandbox) to install the controller.
For example, to install the latest core components:

```bash
# Replace "v0.1.0" with the actual desired version
export VERSION="v0.1.0"
kubectl apply -f https://github.com/kubernetes-sigs/agent-sandbox/releases/download/${VERSION}/manifest.yaml
```

## Step 3: Deploy Mutating Admission Policy (Advanced)

This section explains how to configure a `MutatingAdmissionPolicy` (using CEL) to automatically rewrite the spec of pods with `runtimeClassName: sandboxvm` to use the `pause` container and remove all else.

```bash
kubectl apply -f manifests/
```

## Step 4: Deploy Test Sandoxes to see pass through and mutation

Here are examples of a `Sandbox` resource.

### Example A: Sandbox WITHOUT runtimeClass (Admitted as is)

This sandbox targets a runtime class other than `sandboxvm` (or none), so it should be admitted without mutation.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: agents.x-k8s.io/v1alpha1
kind: Sandbox
metadata:
  name: test-debian-sandbox
spec:
  podTemplate:
    metadata:
      labels:
        app: test-debian
    spec:
      containers:
      - name: debian-container
        image: debian:latest
        command: ["/bin/bash", "-c"]
        args: ["echo 'hello from the agent' && sleep 3600"]
EOF
```

Verify that the created pod runs and echos the message:
```bash
kubectl logs -l app=test-debian
```

### Example B: Sandbox WITH runtimeClass: sandboxvm (Mutated to Pause)

This sandbox uses `runtimeClassName: sandboxvm` in its pod template, so it should be intercepted and mutated to use a `pause` container.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: agents.x-k8s.io/v1alpha1
kind: Sandbox
metadata:
  name: test-vm-sandbox
spec:
  podTemplate:
    metadata:
      labels:
        app: test-vm
    spec:
      runtimeClassName: sandboxvm
      containers:
      - name: debian-container
        image: debian:latest
        command: ["/bin/bash", "-c"]
        args: ["echo 'hello from the sandboxvm' && sleep 3600"]
EOF
```

Verify that the created pod was mutated! Check its image:
```bash
kubectl get pod -l app=test-vm -o jsonpath='{.items[0].spec.containers[0].image}'; echo
```
You should see `registry.k8s.io/pause:3.10` instead of `debian:latest`.


## Step 5: Clean up

Finally clean up:

```bash
# delete the sandboxes
kubectl delete sandbox test-debian-sandbox test-vm-sandbox

# or just get rid of the whole cluster
kind delete cluster --name=sandbox-cluster
```



