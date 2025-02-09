---
title: 'CKA Practice: Upgrade Multi-Node Kubernetes Cluster'

description: |
  This exercise tests your ability to safely upgrade a multi-node Kubernetes cluster from version 1.30 to 1.31 following the standard upgrade procedure.

kind: challenge

playground:
  name: mini-lan-ubuntu

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

#
# All of these init tasks are directly in the challenge markdown because it's not currently possible to reference a custom
# playground when creating a challenge. The current accepted solution is to use the public playground and add init tasks
# on top of it to add any customizations.
#
  init_node_01_configure_control_plane:
      name: init_node_01_configure_control_plane
      machine: node-01
      init: true
      user: root
      timeout_seconds: 300
      needs:
          - init_node_01_install_k8s
          - init_node_01_configure_networking
          - init_node_01_configure_routes
      run: "#!/bin/bash\nset -e\n\nIP_ADDRESS=$(ifconfig | grep -A 2 eth0 | grep \"inet \" | awk '{print $2}')\n\nPOD_CIDR=\"192.168.0.0/16\"\n\nkubeadm init --ignore-preflight-errors=NumCPU --apiserver-advertise-address=$IP_ADDRESS --apiserver-cert-extra-sans=$IP_ADDRESS --pod-network-cidr=$POD_CIDR --node-name \"control-01\"\n\nmkdir -p $HOME/.kube\ncp -i /etc/kubernetes/admin.conf $HOME/.kube/config\nchown $(id -u):$(id -g) $HOME/.kube/config\n\n### and here is where we run the join command via ssh\nJOIN_COMMAND=$(kubeadm token create --print-join-command)\n\nfor node in node-02 node-03; do\n\n    echo \"Waiting for /tmp/all_done on $node...\"\n    while ! ssh $node \"screen -d -m test -f /tmp/all_done\"; do\n        sleep 5\n        echo \"Still waiting for /tmp/all_done on $node...\"\n    done\n    \n    echo \"File found on $node, proceeding with join command...\"\n\n    case $node in\n        \"node-02\")\n            ssh $node -f \"screen -d -m $JOIN_COMMAND --node-name 'worker-01'\"\n            ;;\n        \"node-03\")\n            ssh $node -f \"screen -d -m $JOIN_COMMAND --node-name 'worker-02'\"\n            ;;\n    esac\ndone\n\n"
  init_node_01_configure_networking:
      name: init_node_01_configure_networking
      machine: node-01
      init: true
      user: root
      timeout_seconds: 300
      needs:
          - init_node_01_install_k8s
      run: |
          #!/bin/bash
          set -e

          cat > /etc/cni/net.d/99-loopback.conf <<EOF
          {
            "cniVersion": "1.1.0",
            "name": "lo",
            "type": "loopback"
          }
          EOF

          cat > /etc/cni/net.d/10-bridge.conf <<EOF
          {
            "cniVersion": "1.0.0",
            "name": "bridge",
            "type": "bridge",
            "bridge": "cni0",
            "isGateway": true,
            "ipMasq": true,
            "ipam": {
              "type": "host-local",
              "ranges": [
                [{"subnet": "10.244.0.0/24"}]
              ],
              "routes": [{"dst": "0.0.0.0/0"}]
            }
          }
          EOF
  init_node_01_configure_routes:
      name: init_node_01_configure_routes
      machine: node-01
      init: true
      user: root
      timeout_seconds: 300
      needs:
          - init_node_01_install_k8s
      run: |-
          ip route add 10.244.1.0/24 via 172.16.0.3  # worker-01
          ip route add 10.244.2.0/24 via 172.16.0.4  # worker-02
  init_node_01_configure_ssh:
      name: init_node_01_configure_ssh
      machine: node-01
      init: true
      user: root
      timeout_seconds: 300
      run: |
          #!/bin/bash
          set -e

          cat > ~/.ssh/config <<EOF
          Host *
            StrictHostKeyChecking no
          EOF
  init_node_01_install_k8s:
      name: init_node_01_install_k8s
      machine: node-01
      init: true
      user: root
      timeout_seconds: 300
      run: |-
          #!/bin/bash
          set -e

          cat <<EOF | tee /etc/modules-load.d/k8s.conf
          overlay
          br_netfilter
          EOF

          modprobe overlay
          modprobe br_netfilter

          # sysctl params required by setup, params persist across reboots
          cat <<EOF | tee /etc/sysctl.d/k8s.conf
          net.bridge.bridge-nf-call-iptables  = 1
          net.bridge.bridge-nf-call-ip6tables = 1
          net.ipv4.ip_forward                 = 1
          EOF

          # Apply sysctl params without reboot
          sysctl --system

          swapoff -a

          apt-get update -y
          apt-get install -y software-properties-common gpg curl apt-transport-https ca-certificates screen

          curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/Release.key |  gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg

          echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/ /" | tee /etc/apt/sources.list.d/cri-o.list

          apt-get update -y
          apt-get install -y cri-o

          systemctl daemon-reload
          systemctl enable crio --now
          systemctl start crio.service


          VERSION="v1.30.0"
          wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
          tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
          rm -f crictl-$VERSION-linux-amd64.tar.gz


          KUBERNETES_VERSION=1.30

          mkdir -p /etc/apt/keyrings
          curl -fsSL https://pkgs.k8s.io/core:/stable:/v$KUBERNETES_VERSION/deb/Release.key |  gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
          echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v$KUBERNETES_VERSION/deb/ /" | tee /etc/apt/sources.list.d/kubernetes.list

          apt-get update -y

          apt-get install -y kubelet kubeadm kubectl

          apt-mark hold kubelet kubeadm kubectl

          apt-get install -y jq
          local_ip="$(ip --json addr show eth0 | jq -r '.[0].addr_info[] | select(.family == "inet") | .local')"

          cat << EOF | tee /etc/default/kubelet
          KUBELET_EXTRA_ARGS=--node-ip=$local_ip
          EOF
  init_node_02_configure_networking:
      name: init_node_02_configure_networking
      machine: node-02
      init: true
      user: root
      timeout_seconds: 300
      needs:
          - init_node_02_install_k8s
      run: |
          #!/bin/bash
          set -e

          cat > /etc/cni/net.d/99-loopback.conf <<EOF
          {
            "cniVersion": "1.1.0",
            "name": "lo",
            "type": "loopback"
          }
          EOF

          cat > /etc/cni/net.d/10-bridge.conf <<EOF
          {
            "cniVersion": "1.0.0",
            "name": "bridge",
            "type": "bridge",
            "bridge": "cni0",
            "isGateway": true,
            "ipMasq": true,
            "ipam": {
              "type": "host-local",
              "ranges": [
                [{"subnet": "10.244.1.0/24"}]
              ],
              "routes": [{"dst": "0.0.0.0/0"}]
            }
          }
          EOF
  init_node_02_configure_routes:
      name: init_node_02_configure_routes
      machine: node-02
      init: true
      user: root
      timeout_seconds: 300
      needs:
          - init_node_02_install_k8s
      run: |
          ip route add 10.244.0.0/24 via 172.16.0.2  # cplane
          ip route add 10.244.2.0/24 via 172.16.0.4  # worker-02
  init_node_02_install_k8s:
      name: init_node_02_install_k8s
      machine: node-02
      init: true
      user: root
      timeout_seconds: 300
      run: |-
          #!/bin/bash
          set -e

          cat <<EOF | tee /etc/modules-load.d/k8s.conf
          overlay
          br_netfilter
          EOF

          modprobe overlay
          modprobe br_netfilter

          # sysctl params required by setup, params persist across reboots
          cat <<EOF | tee /etc/sysctl.d/k8s.conf
          net.bridge.bridge-nf-call-iptables  = 1
          net.bridge.bridge-nf-call-ip6tables = 1
          net.ipv4.ip_forward                 = 1
          EOF

          # Apply sysctl params without reboot
          sysctl --system

          swapoff -a

          apt-get update -y
          apt-get install -y software-properties-common gpg curl apt-transport-https ca-certificates screen

          curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/Release.key |  gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg

          echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/ /" | tee /etc/apt/sources.list.d/cri-o.list

          apt-get update -y
          apt-get install -y cri-o

          systemctl daemon-reload
          systemctl enable crio --now
          systemctl start crio.service


          VERSION="v1.30.0"
          wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
          tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
          rm -f crictl-$VERSION-linux-amd64.tar.gz


          KUBERNETES_VERSION=1.30

          mkdir -p /etc/apt/keyrings
          curl -fsSL https://pkgs.k8s.io/core:/stable:/v$KUBERNETES_VERSION/deb/Release.key |  gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
          echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v$KUBERNETES_VERSION/deb/ /" | tee /etc/apt/sources.list.d/kubernetes.list

          apt-get update -y

          apt-get install -y kubelet kubeadm kubectl

          apt-mark hold kubelet kubeadm kubectl

          apt-get install -y jq
          local_ip="$(ip --json addr show eth0 | jq -r '.[0].addr_info[] | select(.family == "inet") | .local')"

          cat << EOF | tee /etc/default/kubelet
          KUBELET_EXTRA_ARGS=--node-ip=$local_ip
          EOF
  init_node_02_write_success_file:
      name: init_node_02_write_success_file
      machine: node-02
      init: true
      user: root
      timeout_seconds: 300
      needs:
          - init_node_02_configure_networking
          - init_node_02_configure_routes
          - init_node_02_install_k8s
      run: |
          #!/bin/bash
          set -e

          echo "done" >> /tmp/all_done
  init_node_03_configure_networking:
      name: init_node_03_configure_networking
      machine: node-03
      init: true
      user: root
      timeout_seconds: 300
      needs:
          - init_node_03_install_k8s
      run: |
          #!/bin/bash
          set -e

          cat > /etc/cni/net.d/99-loopback.conf <<EOF
          {
            "cniVersion": "1.1.0",
            "name": "lo",
            "type": "loopback"
          }
          EOF

          cat > /etc/cni/net.d/10-bridge.conf <<EOF
          {
            "cniVersion": "1.0.0",
            "name": "bridge",
            "type": "bridge",
            "bridge": "cni0",
            "isGateway": true,
            "ipMasq": true,
            "ipam": {
              "type": "host-local",
              "ranges": [
                [{"subnet": "10.244.2.0/24"}]
              ],
              "routes": [{"dst": "0.0.0.0/0"}]
            }
          }
          EOF
  init_node_03_configure_routes:
      name: init_node_03_configure_routes
      machine: node-03
      init: true
      user: root
      timeout_seconds: 300
      needs:
          - init_node_03_install_k8s
      run: |-
          ip route add 10.244.0.0/24 via 172.16.0.2  # cplane
          ip route add 10.244.1.0/24 via 172.16.0.3  # worker-01
  init_node_03_install_k8s:
      name: init_node_03_install_k8s
      machine: node-03
      init: true
      user: root
      timeout_seconds: 300
      run: |-
          #!/bin/bash
          set -e

          cat <<EOF | tee /etc/modules-load.d/k8s.conf
          overlay
          br_netfilter
          EOF

          modprobe overlay
          modprobe br_netfilter

          # sysctl params required by setup, params persist across reboots
          cat <<EOF | tee /etc/sysctl.d/k8s.conf
          net.bridge.bridge-nf-call-iptables  = 1
          net.bridge.bridge-nf-call-ip6tables = 1
          net.ipv4.ip_forward                 = 1
          EOF

          # Apply sysctl params without reboot
          sysctl --system

          swapoff -a

          apt-get update -y
          apt-get install -y software-properties-common gpg curl apt-transport-https ca-certificates screen

          curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/Release.key |  gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg

          echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/ /" | tee /etc/apt/sources.list.d/cri-o.list

          apt-get update -y
          apt-get install -y cri-o

          systemctl daemon-reload
          systemctl enable crio --now
          systemctl start crio.service


          VERSION="v1.30.0"
          wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
          tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
          rm -f crictl-$VERSION-linux-amd64.tar.gz


          KUBERNETES_VERSION=1.30

          mkdir -p /etc/apt/keyrings
          curl -fsSL https://pkgs.k8s.io/core:/stable:/v$KUBERNETES_VERSION/deb/Release.key |  gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
          echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v$KUBERNETES_VERSION/deb/ /" | tee /etc/apt/sources.list.d/kubernetes.list

          apt-get update -y

          apt-get install -y kubelet kubeadm kubectl

          apt-mark hold kubelet kubeadm kubectl

          apt-get install -y jq
          local_ip="$(ip --json addr show eth0 | jq -r '.[0].addr_info[] | select(.family == "inet") | .local')"

          cat << EOF | tee /etc/default/kubelet
          KUBELET_EXTRA_ARGS=--node-ip=$local_ip
          EOF
  init_node_03_write_success_file:
      name: init_node_03_write_success_file
      machine: node-03
      init: true
      user: root
      timeout_seconds: 300
      needs:
          - init_node_03_configure_networking
          - init_node_03_configure_routes
          - init_node_03_install_k8s
      run: |
        #!/bin/bash
        set -e

        echo "done" >> /tmp/all_done

# 
# These are where the actual challenge tasks start. Everything above is just setting up the k8s cluster
# 
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

  verify_worker_kubeadm_upgrade_node02:
    machine: node-02
    needs:
      - verify_worker_kubeadm_node02
    run: |
      if [ -d "/etc/kubernetes/tmp" ]; then
        echo "The directory /etc/kubernetes/tmp exists."
        exit 0
      else
        echo "The directory /etc/kubernetes/tmp does not exist."
        exit 1
      fi


  verify_worker_kubeadm_upgrade_node03:
    machine: node-03
    needs:
      - verify_worker_kubeadm_node03
    run: |
      if [ -d "/etc/kubernetes/tmp" ]; then
        echo "The directory /etc/kubernetes/tmp exists."
        exit 0
      else
        echo "The directory /etc/kubernetes/tmp does not exist."
        exit 1
      fi

  verify_worker_components_node02:
    machine: node-02
    needs:
      - verify_worker_kubeadm_upgrade_node02
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
      - verify_worker_kubeadm_upgrade_node03
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
Follow the guide in the [official docs](https://v1-31.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/upgrading-linux-nodes/).
::

::hint-box
---
:summary: Hint 2
---
If you need help adding the new minor version package repository, read [these docs](https://v1-31.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/#switching-to-another-kubernetes-package-repository).
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
You can follow the steps [here](https://v1-31.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#upgrading-control-plane-nodes)
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
The steps to upgrade the components (kubelet and kubectl) are [here](https://v1-31.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#upgrade-kubelet-and-kubectl).
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
The process to upgrade each worker node is [here](https://v1-31.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/upgrading-linux-nodes/).
::

Upgrade the worker node-02 itself:
::simple-task
---
:tasks: tasks
:name: verify_worker_kubeadm_upgrade_node02
---
#active
Verifying node upgrade...

#completed
Node-02 has been successfully upgraded
::

Upgrade the worker node-03 itself:
::simple-task
---
:tasks: tasks
:name: verify_worker_kubeadm_upgrade_node03
---
#active
Verifying node upgrade...

#completed
Node-03 has been successfully upgraded
::

::hint-box
---
:summary: Hint 6
---
Remember to run `kubeadm upgrade node` instead of `kubeadm upgrade plan`.
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
:summary: Hint 7
---
Remember to restart the system-daemon and the kubelet
```
systemctl daemon-reload
systemctl restart kubelet
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
:summary: Hint 8
---
For each worker node, you can follow the steps [here](https://v1-31.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/upgrading-linux-nodes/#upgrade-kubelet-and-kubectl).
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
:summary: Hint 9
---
Verify cluster health with:
```bash
kubectl get nodes
kubectl get pods -A
kubectl get --raw='/readyz?verbose'
```
Make sure all nodes are Ready and all pods are Running.
::
