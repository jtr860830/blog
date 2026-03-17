---
title: 更新由 kubeadm 部署的 K8s 叢集設定
date: 2026-03-17 22:57:03
categories:
  - Kubernetes
tags:
  - Cloud Native
  - Kubernetes
  - Linux
---

[![hackmd-github-sync-badge](https://hackmd.io/aIbZfz1TTnG68-nilrsqxQ/badge)](https://hackmd.io/aIbZfz1TTnG68-nilrsqxQ)

由 `kubeadm` 產生的 Kubernetes 叢集會將叢集的設定值以 ConfigMap 的形式儲存在 `kube-system` 這個 namespace 底下，可以使用以下指令觀察到:

```
tux@kubeadm-m1:~$ kubectl get -n kube-system configmap
NAME                                                   DATA   AGE
coredns                                                1      4h
extension-apiserver-authentication                     6      4h
kube-apiserver-legacy-service-account-token-tracking   1      4h
kube-proxy                                             2      4h
kube-root-ca.crt                                       1      4h
kubeadm-config                                         1      4h
kubelet-config                                         1      4h
```

叢集的設定值就是存在 `kubeadm-config` 這個 ConfigMap 中，接下來以更新 etcd 的命令列參數為範例，演示更新的步驟。

# 輸出目前的設定檔

可以使用以下指令去輸出設定檔:

`kubectl get -n kube-system configmap kubeadm-config -o jsonpath={.data.ClusterConfiguration} > config.yaml`

> 不使用 `-o yaml` 的原因是 `kubeadm` 只吃完整 ConfigMap 中的 data > ClusterConfiguration 的內容

設定檔應該會長的類似這樣:

```yaml
apiServer: {}
apiVersion: kubeadm.k8s.io/v1beta4
caCertificateValidityPeriod: 87600h0m0s
certificateValidityPeriod: 8760h0m0s
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: 192.168.11.30:6443
controllerManager: {}
dns: {}
encryptionAlgorithm: RSA-2048
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.k8s.io
kind: ClusterConfiguration
kubernetesVersion: v1.35.2
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
proxy: {}
scheduler: {}
```

# 修改設定

可以看到大部分核心元件的設定都會放在這邊，在這邊找到 etcd 的區塊加上下方整個 `extraArgs:` 的部分，目的是更新 etcd metrics server 監聽的位置

```yaml
etcd:
  local:
    dataDir: /var/lib/etcd
    extraArgs:
    - name: listen-metrics-urls
      value: http://0.0.0.0:2381
```

# 套用設定

1. `sudo kubeadm init phase upload-config kubeadm --config ./config.yaml`
2. `sudo kubeadm init phase etcd local --config ./config.yaml`

使用這兩個命令就會套用更新內容了，第一個命令會先把更新套用在 ConfigMap 中的設定值，然後第二個命令更新存在 `/etc/kubernetes/manifests/` 中的 `etcd.yaml` 檔案。

比較常見的情況，如果只使用 `kubectl edit configmap` 會沒更新到本地 `/etc/kubernetes/manifests/etcd.yaml` 的設定；而如果直接更新 `/etc/kubernetes/manifests/etcd.yaml` 又會沒有更新到 ConfigMap，而導致之後更新叢集元件參數不如預期的情況。

而使用這些命令需要注意: 存在 `/etc/kubernetes/manifests/` 中的 static Pods 設定更新，只會套用在當前使用這個命令的節點，而 etcd 設定只會在 Control Plane 節點上，所以在多個 Control Plane 節點的情況下，需要在每個 Control Plane 節點都使用一次。

最後筆者這邊是示範更新 etcd 參數的情況，如果是更新其他元件需要去看一下 `kubeadm` 有哪些 `init phase` 或是 `upgrade phasee` 可用，然後決定適合的 phase 去執行，可以參考:

- [kubeadm init phase](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init-phase/)
- [kubeadm upgrade phase](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-upgrade-phase/)
- [Kubeadm 如何永久套用 components 參數](https://hackmd.io/@wu-andy/BJpvBQGBle)
