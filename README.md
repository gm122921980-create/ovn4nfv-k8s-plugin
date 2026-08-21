# OVN4NFV K8s Plugin - Network controller
This plugin addresses the below requirements, for networking
workloads as well typical application workloads
- Multi ovn network support
- Multi-interface ovn support
- Multi-IP address support
- Dynamic creation of virtual networks
- Route management across virtual networks and external networks
- Service Function chaining(SFC) support in Kubernetes
- SRIOV Overlay networking (WIP)
- OVN load balancer (WIP)

## How it works

OVN4NFV consist of 4 major components
- OVN control plane
- OVN controller
- Network Function Network(NFN) k8s operator/controller
- Network Function Network(NFN) agent

OVN control plane and OVN controller take care of OVN configuration and installation in each node in Kubernetes. NFN operator runs in the Kubernetes master and NFN agent run as a daemonset in each node.

### OVN4NFV architecture blocks
![ovn4nfv k8s arc block](./images/ovn4nfv-k8s-arch-block.png)

#### NFN Operator
* Exposes virtual, provider, chaining CRDs to external world
* Programs OVN to create L2 switches
* Watches for PODs being coming up
 * Assigns IP addresses for every network of the deployment
 * Looks for replicas and auto create routes for chaining to work
 * Create LBs for distributing the load across CNF replicas
#### NFN Agent
* Performs CNI operations.
* Configures VLAN and Routes in Linux kernel (in case of routes, it could do it in both root and network namespaces)
* Communicates with OVSDB to inform of provider interfaces. (creates ovs bridge and creates external-ids:ovn-bridge-mappings)

### Networks traffice between pods
![ovn4nfv network traffic](./images/ovn4nfv-network-traffic.png)

ovn4nfv-default-nw is the default logic switch create for the default networking in kubernetes pod network for cidr 10.233.64.0/18. Both node and pod in the kubernetes cluster share the same ipam information.

### Service Function Chaining Demo
![sfc-with-sdewan](./images/sfc-with-sdewan.png)

In general production env, we have multiple Network function such as SLB, NGFW and SDWAN CNFs.

There are general 3 sfc flows are there:
* Packets from the pod to reach internet: Ingress (L7 LB) -> SLB -> NGFW -> SDWAN CNF -> External router -> Internet
* Packets from the pod to internal server in the corp network: Ingress (L7 LB) -> SLB -> M3 server
* Packets from the internal server M3 to reach internet: M3 -> SLB -> NGFW -> SDWAN CNF -> External router -> Internet

OVN4NFV SFC currently support all 3 follows. The detailed demo is include [demo/sfc-setup/README.md](./demo/sfc-setup/README.md)

# Quickstart Installation Guide
### kubeadm

Install the [docker](https://docs.docker.com/engine/install/ubuntu/) in the Kubernetes cluster node.
Follow the steps in [create cluster kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/) to create kubernetes cluster in master
In the master node run the `kubeadm init` as below. The ovn4nfv uses pod network cidr `10.233.64.0/18`
```
    $ kubeadm init --kubernetes-version=1.19.0 --pod-network-cidr=10.233.64.0/18 --apiserver-advertise-address=<master_eth0_ip_address>
```
Ensure the master node taint for no schedule is removed and labelled with `ovn4nfv-k8s-plugin=ovn-control-plane`
```
nodename=$(kubectl get node -o jsonpath='{.items[0].metadata.name}')
kubectl taint node $nodename node-role.kubernetes.io/master:NoSchedule-
kubectl label --overwrite node $nodename ovn4nfv-k8s-plugin=ovn-control-plane
```
Deploy the ovn4nfv Pod network to the cluster.
```
    $ kubectl apply -f deploy/ovn-daemonset.yaml
    $ kubectl apply -f deploy/ovn4nfv-k8s-plugin.yaml
```
Join worker node by running the `kubeadm join` on each node as root as mentioned in [create cluster kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

### kubespray

Kubespray support the ovn4nfv as the network plugin- please follow the steps in [kubernetes-sigs/kubespray//docs/ovn4nfv.md](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/ovn4nfv.md)

## Comprehensive Documentation

- [How to use](doc/how-to-use.md)
- [Configuration](doc/configuration.md)
- [Development](doc/development.md)

## Contact Us

For any questions about ovn4nfv k8s , feel free to ask a question in #general in the [ICN slack](https://akraino-icn-admin.herokuapp.com/), or open up a https://jira.opnfv.org/issues/.


/**
 * Copyright © 2026 徐嘉糧 (GUBON LUCID OS / GUBON-EX). All rights reserved.
 *
 * 中文：
 * 本系統之原始碼、系統架構、軟體設計、演算法邏輯、資料結構、
 * 私有化簽章與驗證機制，以及相關商業流程與閉環設計，
 * 其依法可受保護之權利，除另有明確書面約定外，均由權利人享有。
 *
 * English:
 * The source code, system architecture, software design, algorithmic logic,
 * data structures, sovereign signing and verification mechanisms, and
 * related commercial workflows and closed-loop designs of this system,
 * together with all rights legally protectable therein, are owned by the
 * rights holder unless otherwise expressly agreed in writing.
 *
 * Unauthorized reproduction, distribution, modification, disclosure,
 * sublicensing, or deployment is prohibited to the extent permitted by law.
 */
# Intellectual Property & Sovereign Notice

Copyright © 2026 徐嘉糧  
GUBON LUCID OS / GUBON-EX  
All rights reserved.

## 中文

除另有明確書面授權或契約約定外，GUBON LUCID OS 及 GUBON-EX 所涉及之原始碼、軟體架構、系統設計、演算法與程式邏輯、資料結構、私有化簽章及驗證機制、商業流程、決策流程、產品設計及相關技術文件，其依法可受保護之智慧財產權及其他權利均由權利人享有。

未經適當授權，任何人不得對受保護內容進行未經授權之複製、重製、修改、散布、公開傳輸、轉讓、再授權、商業利用或部署。

本聲明不影響第三方軟體、開源元件、API、SDK、模型、服務或其他內容所適用之原有授權條款。

---

## English

Unless expressly licensed or otherwise agreed in writing, all legally protectable intellectual property and other rights relating to GUBON LUCID OS and GUBON-EX, including source code, software architecture, system design, algorithms and program logic, data structures, sovereign signing and verification mechanisms, commercial workflows, decision processes, product designs, and related technical documentation, are owned by the rights holder.

Without appropriate authorization, no person may reproduce, modify, distribute, publicly communicate, transfer, sublicense, commercially exploit, or deploy protected materials.

Nothing in this notice overrides the applicable license terms of third-party software, open-source components, APIs, SDKs, models, services, or other third-party materials.

# Third-Party & Open Source Notices

GUBON LUCID OS / GUBON-EX incorporates various open-source software, third-party libraries, APIs, and SDKs (including but not limited to Node.js, React, Next.js, PostgreSQL, Prisma, Redis, BullMQ, and payment SDKs such as PayPal REST API v2). 

Each third-party component remains subject to its respective original license terms (e.g., MIT, Apache 2.0, ISC). Nothing in the GUBON-EX proprietary licensing structure alters, supersedes, or restricts the rights and obligations granted under those respective open-source or third-party licenses.

For detailed third-party dependency licenses, please refer to the respective package manifests (`package.json`) and upstream documentation.
# Intellectual Property & Sovereign Notice

Copyright © 2026 徐嘉糧
GUBON LUCID OS / GUBON-EX
All rights reserved.

## 中文

除另有明確書面授權或契約約定外，GUBON LUCID OS
及 GUBON-EX 所涉及之原始碼、軟體架構、系統設計、
演算法與程式邏輯、資料結構、私有化簽章及驗證機制、
商業流程、決策流程、產品設計及相關技術文件，
其依法可受保護之智慧財產權及其他權利均由權利人享有。

未經適當授權，任何人不得對受保護內容進行未經授權之
複製、重製、修改、散布、公開傳輸、轉讓、再授權、
商業利用或部署。

本聲明不影響第三方軟體、開源元件、API、SDK、模型、
服務或其他內容所適用之原有授權條款。

## English

Unless expressly licensed or otherwise agreed in writing,
all legally protectable intellectual property and other rights
relating to GUBON LUCID OS and GUBON-EX, including source code,
software architecture, system design, algorithms and program logic,
data structures, sovereign signing and verification mechanisms,
commercial workflows, decision processes, product designs,
and related technical documentation, are owned by the rights holder.

Without appropriate authorization, no person may reproduce,
modify, distribute, publicly communicate, transfer, sublicense,
commercially exploit, or deploy protected materials.

Nothing in this notice overrides the applicable license terms
of third-party software, open-source components, APIs, SDKs,
models, services, or other third-party materials.

eagle19900203@gmail.com
gubonlucid.com