# New install in Kia の Lab

## RHEL 9.3 部署 Kubernetes 1.32（含 Flannel、Metrics Server、Dashboard）


### 系統需求

✅ RHEL 9.3（Master 與 Worker 節點）\
✅ Kubernetes 1.32\
✅ `containerd` 作為容器運行時\
✅ `Flannel`（網路插件）\
✅ `Metrics Server`（監控）\
✅ `Kubernetes Dashboard`（管理界面）

---

## **步驟 1：關閉 Swap 及 SELinux ＆ 網路前置作業**

```bash
# 關閉 Swap
sudo swapoff -a

# 永久禁用 Swap
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# 關閉 SELinux
sudo setenforce 0

# 永久禁用 SELinux
sudo sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

＃ 關閉防火牆
systemctl stop firewalld
systemctl disable firewalld

cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system

yum install -y ipvsadm conntrack sysstat curl

modprobe br_netfilter && modprobe ip_vs

## 無法 ssh login 解法
[root@master-01 ~]# yum -y install  openssh-server

[root@master-01 ~]# rpm -qa | grep openss
openssh-8.7p1-43.el9.x86_64
openssh-server-8.7p1-43.el9.x86_64
openssh-clients-8.7p1-43.el9.x86_64
openssl-libs-3.2.2-6.el9_5.x86_64
openssl-3.2.2-6.el9_5.x86_64



```

---

## **步驟 2：安裝 containerd**

```bash
# 安裝 containerd
yum-config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo yum install -y containerd.io

# 建立預設配置
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = false/' /etc/containerd/config.toml
# ref https://kubernetes.io/zh-cn/docs/setup/production-environment/container-runtimes/#cgroup-drivers

# 啟用 containerd
sudo systemctl enable --now containerd

＃ 安裝 podman
sudo yum install -y podman

```

---

## **步驟 3：安裝 `kubeadm`, `kubelet`, `kubectl`**

```bash
# 添加 Kubernetes 倉庫
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/repodata/repomd.xml.key
EOF

# 安裝 Kubernetes 套件
sudo yum install -y kubelet kubeadm kubectl

# 啟用 kubelet
sudo systemctl enable --now kubelet
```

---

## **步驟 4：初始化 Kubernetes 集群**

```bash
# 設定 Master 節點 IP（請換成實際 IP）
MASTER_IP="192.168.1.100"

# 初始化 Kubernetes
sudo kubeadm init \
#  --apiserver-advertise-address=$MASTER_IP \
  --apiserver-advertise-address=0.0.0.0 \     # 測試用
  --pod-network-cidr=100.64.0.0/10 \
  --service-cluster-ip-range=10.96.0.0/22


```

初始化完成後，執行以下指令讓 `kubectl` 可用：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## **步驟 5：安裝 Flannel（網路插件）**

```bash
# 下載 Flannel YAML 配置檔
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

**檢查 Flannel 是否成功部署**

```bash
kubectl get pods -n kube-flannel
```

---

## **步驟 6：加入 Worker 節點**

在 **Master 節點** 執行：

```bash
kubeadm token create --print-join-command
```

這將輸出類似：

```bash
kubeadm join 192.168.1.100:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

然後，在 **Worker 節點** 執行：

```bash
sudo kubeadm join 192.168.1.100:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

---

## **步驟 7：安裝 Metrics Server（監控）**

```bash
# 安裝 Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

**驗證 Metrics Server 是否正常運作**

```bash
kubectl get deployment metrics-server -n kube-system
```

---

## **步驟 8：安裝 Kubernetes Dashboard（管理界面）**

```bash
# 安裝 Dashboard
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

### **建立管理員帳戶**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF
```

**取得登入 Token**

```bash
kubectl -n kubernetes-dashboard create token admin-user
```

**啟動 Dashboard**

```bash
kubectl proxy
```

然後在瀏覽器中開啟：

```
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

輸入剛剛取得的 Token，即可登入 **Kubernetes Dashboard**。


---
## **環境設定**
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

## **常用指令**
* kubectl get nodes
* kubectl get nodes -o wide
* kubectl get nodes
* kubectl get pods --all-namespaces
* kubectl get nodes
* kubectl get nodes -o wide

---

是的，這些步驟在 **RHEL 9.3 + Kubernetes 1.31.7** 安裝過程中 **是必要的**，主要用於**網路與負載平衡功能的開啟**。下面是每個指令的作用與必要性：  

---

### **1️⃣ `sysctl --system`**
🔹 **作用：**  
- 重新載入所有 `/etc/sysctl.d/` 內的 **系統內核參數（kernel parameters）** 設定。  
- 確保 Kubernetes 相關的網路轉發與封包處理規則生效，例如 `net.bridge.bridge-nf-call-iptables=1`。  

🔹 **是否需要？** ✅ **需要**  
- Kubernetes 需要內核參數 `net.bridge.bridge-nf-call-iptables=1` 來讓 `iptables` 正確處理流量，這步驟確保所有參數生效。

---

### **2️⃣ `yum install -y ipvsadm conntrack sysstat curl`**
🔹 **作用：**
- **`ipvsadm`**：管理 **IPVS（IP 虛擬伺服器）**，Kubernetes 預設使用 **IPVS 模式的 kube-proxy** 來進行負載均衡。  
- **`conntrack`**：監控與管理網路連線追蹤（connection tracking），Kubernetes 需要它來處理封包與 NAT。  
- **`sysstat`**：提供 `sar` 等系統監控工具，可用於檢查 CPU、記憶體與網路狀況（可選，但推薦）。  
- **`curl`**：用來測試 HTTP API，Kubernetes 安裝與診斷時經常需要（例如 `kubectl get` 相關 API 測試）。  

🔹 **是否需要？** ✅ **需要**  
- Kubernetes 使用 `ipvsadm` 來管理內部 **Service 負載均衡**，如果沒裝，可能會導致 `kube-proxy` 運行失敗。  
- `conntrack` 必須安裝，否則 `kube-proxy` 可能無法正常追蹤封包狀態。  

---

### **3️⃣ `modprobe br_netfilter && modprobe ip_vs`**
🔹 **作用：**
- **`modprobe br_netfilter`**  
  - 讓 Linux 內核的 **bridge-nf（橋接網路封包過濾）** 模組啟動，確保 Kubernetes 能夠正確處理跨節點的網路流量。  
  - 這與 `net.bridge.bridge-nf-call-iptables=1` 有關，缺少它可能導致 **Pod 之間的網路封包無法正確轉發**。  

- **`modprobe ip_vs`**  
  - 載入 **IPVS 負載均衡模組**，確保 `kube-proxy` 能夠運行在 **IPVS 模式**，提高 Kubernetes Service 負載均衡的效能。  
  - Kubernetes 會根據需求載入 `ip_vs_rr`、`ip_vs_wrr`、`ip_vs_sh` 等模組，以支援不同的負載均衡策略。  

🔹 **是否需要？** ✅ **需要**  
- 沒有 `br_netfilter`，**iptables 可能無法處理 Pod 之間的流量**，影響 Kubernetes 網路。  
- 沒有 `ip_vs`，**Kubernetes 可能會退回到 `iptables` 模式**，導致效能降低。  

---

### **🚀 總結**
| 指令 | 是否需要 | 作用 |
| --- | --- | --- |
| `sysctl --system` | ✅ **需要** | 重新載入 `sysctl` 設定，確保網路參數生效 |
| `yum install -y ipvsadm conntrack sysstat curl` | ✅ **需要** | 安裝 **IPVS、連線追蹤、系統監控與測試工具** |
| `modprobe br_netfilter && modprobe ip_vs` | ✅ **需要** | 啟用 **橋接網路與 IPVS 負載均衡**，確保 `kube-proxy` 運行 |

這些步驟在 **RHEL 9.3 + Kubernetes 1.31.7** 環境下都是必要的，請務必執行！🚀

---


## **結論**

✅ **符合 Kubernetes 1.32，適用於 RHEL 9.3**\
✅ **完整安裝 `Flannel`（網路）、`Metrics Server`（監控）、`Dashboard`（管理）**\
✅ **已設定 `service-cluster-ip-range=10.96.0.0/22` 和 `cluster-cidr=100.64.0.0/10`**\
✅ **提供完整的 Worker 加入方式與測試指令**\
✅ **新增關閉 Swap 及 SELinux 步驟，確保 Kubernetes 正常運作**






