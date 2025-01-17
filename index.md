---
title: 'CKA Practice: Upgrade Multi-Node Kubernetes Cluster'

description: |
  This exercise tests your ability to safely upgrade a multi-node Kubernetes cluster from version 1.30 to 1.31 following the standard upgrade procedure.

kind: challenge

playground: custom-18fa3073

playgroundOptions:
  tabs:
  - machine: node-01
  - machine: node-02
  - machine: node-03

  machines:
    - name: node-01
    - name: node-02
    - name: node-03

cover: __static__/kubeadm-upgrade.png

createdAt: 2025-01-10
updatedAt: 2025-01-15

difficulty: easy

categories:
  - kubernetes

tagz:
  - cka

tasks:
  verify_controlplane_kubeadm:
    run: |
      KUBEADM_VERSION=$(kubeadm version -o json | jq -r '.clientVersion.gitVersion')
      if [[ "$KUBEADM_VERSION" == "v1.31."* ]]; then
        echo "Control plane kubeadm upgraded successfully!"
        exit 0
      else
        exit 1
      fi

  verify_controlplane_components:
    needs:
      - verify_controlplane_kubeadm
    run: |
      API_VERSION=$(kubectl get nodes control-01 -o jsonpath='{.status.nodeInfo.kubeletVersion}')
      if [[ "$API_VERSION" == "v1.31."* ]]; then
        echo "Control plane components upgraded successfully!"
        exit 0
      else
        exit 1
      fi


  # we need another check here for the actual cluster upgrade...currently it's all green even 
  # before running `kubeadm upgrade apply v1.31.5

  verify_worker_kubeadm:
    needs:
      - verify_controlplane_components
    run: |
      WORKER1_VERSION=$(ssh root@node-02 'kubeadm version -o json' | jq -r '.clientVersion.gitVersion')
      WORKER2_VERSION=$(ssh root@node-03 'kubeadm version -o json' | jq -r '.clientVersion.gitVersion')
      if [[ "$WORKER1_VERSION" == "v1.31."* ]] && [[ "$WORKER2_VERSION" == "v1.31."* ]]; then
        echo "Worker nodes kubeadm upgraded successfully!"
        exit 0
      else
        exit 1
      fi

  verify_worker_components:
    needs:
      - verify_worker_kubeadm
    run: |
      WORKER1_VERSION=$(kubectl get nodes worker-01 -o jsonpath='{.status.nodeInfo.kubeletVersion}')
      WORKER2_VERSION=$(kubectl get nodes worker-02 -o jsonpath='{.status.nodeInfo.kubeletVersion}')
      if [[ "$WORKER1_VERSION" == "v1.31."* ]] && [[ "$WORKER2_VERSION" == "v1.31."* ]]; then
        echo "Worker nodes components upgraded successfully!"
        exit 0
      else
        exit 1
      fi

  verify_cluster_health:
    needs:
      - verify_worker_components
    timeout_seconds: 30
    run: |
      # Check node status
      NODE_STATUS=$(kubectl get nodes -o jsonpath='{.items[*].status.conditions[?(@.type=="Ready")].status}')
      
      # Check pod status
      POD_STATUS=$(kubectl get pods -A -o jsonpath='{.items[*].status.phase}' | tr ' ' '\n' | sort | uniq)
      
      # Check components status
      COMPONENTS_HEALTH=$(kubectl get componentstatuses -o jsonpath='{.items[*].conditions[0].type}')
      
      if [[ "$NODE_STATUS" == "True True True" ]] && \
         [[ "$POD_STATUS" == "Running" ]] && \
         [[ "$COMPONENTS_HEALTH" == "Healthy Healthy Healthy" ]]; then
        echo "Cluster health verified successfully!"
        exit 0
      else
        exit 1
      fi

---

In this exercise, you will upgrade a multi-node Kubernetes cluster from version 1.30 to version 1.31. You'll need to follow the standard upgrade procedure while ensuring cluster stability throughout the process. Let's begin!

<img src="__static__/kubeadm-upgrade.png" style="margin: 0px auto; max-width: 600px; width: 100%" alt="kubeadm upgrade">

First, upgrade kubeadm on the control plane node:

::simple-task
---
:tasks: tasks
:name: verify_controlplane_kubeadm
---
#active
Checking control plane kubeadm version...

#completed
Great! Control plane kubeadm has been upgraded to version 1.31.
::

::hint-box
---
:summary: Hint 1
---
You'll need to unhold kubeadm first:
```bash
apt-mark unhold kubeadm
```
::


::hint-box
---
:summary: Hint 2
---
You also need to add version 1.31 to your sources list:
```bash
KUBERNETES_VERSION=1.31
mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v$KUBERNETES_VERSION/deb/Release.key |  gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v$KUBERNETES_VERSION/deb/ /" | tee /etc/apt/sources.list.d/kubernetes.list
```
After installing `kubeadm`, then verify with: `kubeadm version`
::

Next, upgrade the control plane components:

::simple-task
---
:tasks: tasks
:name: verify_controlplane_components
---
#active
Verifying control plane components upgrade...

#completed
Excellent! All control plane components are now running version 1.31.
::

::hint-box
---
:summary: Hint 3
---
1. Plan the upgrade:
```bash
kubeadm upgrade plan
```
2. Apply the upgrade:
```bash
kubeadm upgrade apply v1.31.x
```
3. Upgrade kubelet and kubectl:
```bash
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.31.x-* kubectl=1.31.x-*
apt-mark hold kubelet kubectl
systemctl daemon-reload
systemctl restart kubelet
```
::

Now, upgrade kubeadm on the worker nodes:

::simple-task
---
:tasks: tasks
:name: verify_worker_kubeadm
---
#active
Checking worker nodes kubeadm versions...

#completed
Perfect! Worker nodes kubeadm packages have been upgraded.
::

::hint-box
---
:summary: Hint 4
---
On each worker node:
```bash
apt-mark unhold kubeadm
apt-get update && apt-get install -y kubeadm=1.31.x-*
apt-mark hold kubeadm
```
Then verify with: `kubeadm version`
::

Upgrade the worker node components:

::simple-task
---
:tasks: tasks
:name: verify_worker_components
---
#active
Verifying worker nodes component versions...

#completed
Great! All worker nodes are now running version 1.31.
::

::hint-box
---
:summary: Hint 5
---
For each worker node:
1. Drain the node:
```bash
kubectl drain node-0x --ignore-daemonsets
```
2. Upgrade the node:
```bash
kubeadm upgrade node
```
3. Upgrade kubelet and kubectl:
```bash
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.31.x-* kubectl=1.31.x-*
apt-mark hold kubelet kubectl
systemctl daemon-reload
systemctl restart kubelet
```
4. Uncordon the node:
```bash
kubectl uncordon node-0x
```
::

Finally, verify the cluster is healthy:

::simple-task
---
:tasks: tasks
:name: verify_cluster_health
---
#active
Checking overall cluster health...

#completed
Congratulations! The cluster has been successfully upgraded and all components are healthy.
::

::hint-box
---
:summary: Hint 6
---
Verify cluster health with:
```bash
kubectl get nodes
kubectl get pods -A
kubectl get componentstatuses
```
Make sure all nodes are Ready and all pods are Running.
::
