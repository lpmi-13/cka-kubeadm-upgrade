---
title: 'CKA Practice: Upgrade Multi-Node Kubernetes Cluster'

description: |
  This exercise tests your ability to safely upgrade a multi-node Kubernetes cluster from version 1.30 to 1.31 following the standard upgrade procedure.

kind: challenge

playground:
  name: custom-18fa3073

  tabs:
  - machine: node-01
  - machine: node-02
  - machine: node-03

  machines:
    - name: node-01
      users:
        - name: root
          default: true
          welcome: "Welcome to the kubeadm upgrade challenge!\n"
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
    machine: node-01
    run: |
      KUBEADM_VERSION=$(kubeadm version -o json | jq -r '.clientVersion.gitVersion')
      if [[ "$KUBEADM_VERSION" == "v1.31."* ]]; then
        echo "Control plane kubeadm upgraded successfully!"
        exit 0
      else
        exit 1
      fi

  verify_controlplane_upgrade:
    machine: node-01
    needs:
      - verify_controlplane_kubeadm
    run: |
      API_SERVER_VERSION=$(kubectl get pods -n kube-system -l component=kube-apiserver -o jsonpath="{.items[0].spec.containers[0].image}" | cut -d ':' -f 2)

      CONTROLLER_VERSION=$(kubectl get pods -n kube-system -l component=kube-controller-manager -o jsonpath="{.items[0].spec.containers[0].image}" | cut -d ':' -f 2)

      SCHEDULER_VERSION=$(kubectl get pods -n kube-system -l component=kube-scheduler -o jsonpath="{.items[0].spec.containers[0].image}" | cut -d ':' -f 2)

      if [[ "$API_SERVER_VERSION" == "v1.31."* ]] && \
         [[ "$CONTROLLER_VERSION" == "v1.31."* ]] && \
         [[ "$SCHEDULER_VERSION" == "v1.31."* ]]; then
         echo "Control plane components upgraded successfully!"
         exit 0
      else
        exit 1
      fi

  verify_controlplane_components:
    machine: node-01
    needs:
      - verify_controlplane_upgrade
    run: |
      API_VERSION=$(kubectl get nodes control-01 -o jsonpath='{.status.nodeInfo.kubeletVersion}')
      if [[ "$API_VERSION" == "v1.31."* ]]; then
        echo "Control plane components upgraded successfully!"
        exit 0
      else
        exit 1
      fi


  verify_worker_kubeadm_node02:
    machine: node-02
    run: |
      KUBEADM_VERSION=$(kubeadm version -o json | jq -r '.clientVersion.gitVersion')
      if [[ "$KUBEADM_VERSION" == "v1.31."* ]]; then
        echo "Worker node-02 kubeadm upgraded successfully!"
        exit 0
      else
        exit 1
      fi

  verify_worker_kubeadm_node03:
    machine: node-03
    run: |
      KUBEADM_VERSION=$(kubeadm version -o json | jq -r '.clientVersion.gitVersion')
      if [[ "$KUBEADM_VERSION" == "v1.31."* ]]; then
        echo "Worker node-03 kubeadm upgraded successfully!"
        exit 0
      else
        exit 1
      fi

  verify_worker_components_node02:
    machine: node-02
    needs:
      - verify_worker_kubeadm_node02
    run: |
      WORKER2_VERSION=$(kubelet --version | awk '{print $2}')
      if [[ "$WORKER2_VERSION" == "v1.31."* ]]; then
        echo "Worker node-02 components upgraded successfully!"
        exit 0
      else
        exit 1
      fi

  verify_worker_components_node03:
    machine: node-03
    needs:
      - verify_worker_kubeadm_node03
    run: |
      WORKER3_VERSION=$(kubelet --version | awk '{print $2}')
      if [[ "$WORKER3_VERSION" == "v1.31."* ]]; then
        echo "Worker node-03 components upgraded successfully!"
        exit 0
      else
        exit 1
      fi

  verify_all_upgraded:
    machine: node-01
    needs:
      - verify_controlplane_components
    run: |
      VERSIONS=$(kubectl get nodes -o custom-columns=VERSION:.status.nodeInfo.kubeletVersion | tail -n 3 | uniq | wc -l)

      if [[ $VERSIONS == 1 ]]; then
        echo "all nodes updated successfully!"
        exit 0
      else
        exit 1
      fi

  verify_cluster_health:
    machine: node-01
    needs:
      - verify_all_upgraded
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

Next, upgrade the control plane:

::simple-task
---
:tasks: tasks
:name: verify_controlplane_upgrade
---
#active
Verifying control plane upgrade...

#completed
Excellent! The API server, controller manager, and scheduler are now running version 1.31.
::

::hint-box
---
:summary: Hint 3
---
1. Plan the upgrade:
```bash
kubeadm upgrade plan
```

2. Apply the upgrade
```bash
kubeadm upgrade apply v1.31.x
```

3. Verify the upgrade
```bash
kubectl get pods -n kube-system
```

Look for kube-apiserver, kube-controller-manager, and kube-scheduler pods
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
:summary: Hint 4
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
:name: verify_worker_kubeadm_node02
---
#active
Checking worker kubeadm versions on node-02...

#completed
Perfect! Worker node-02 kubeadm package has been upgraded.
::

::simple-task
---
:tasks: tasks
:name: verify_worker_kubeadm_node03
---
#active
Checking worker kubeadm versions on node-03...

#completed
Perfect! Worker node-03 kubeadm package has been upgraded.
::

::hint-box
---
:summary: Hint 5
---
On each worker node:
```bash
apt-mark unhold kubeadm
apt-get update && apt-get install -y kubeadm=1.31.x-*
apt-mark hold kubeadm
```
Then verify with: `kubeadm version`
::

Upgrade the worker node components on node02:

::simple-task
---
:tasks: tasks
:name: verify_worker_components_node02
---
#active
Verifying worker node-02 component versions...

#completed
Great! Worker node-02 components are now running version 1.31.
::

::hint-box
---
:summary: Hint 6
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


Upgrade the worker node components on node03:

::simple-task
---
:tasks: tasks
:name: verify_worker_components_node03
---
#active
Verifying worker node-03 component versions...

#completed
Great! Worker node-03 components are now running version 1.31.
::

::hint-box
---
:summary: Hint 7
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
:summary: Hint 8
---
Verify cluster health with:
```bash
kubectl get nodes
kubectl get pods -A
kubectl get componentstatuses
```
Make sure all nodes are Ready and all pods are Running.
::
