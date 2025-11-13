# 🚀 УСТАНОВКА KUBERNETES С НУЛЯ (официальный способ)

**ОС:** Ubuntu 22.04
**🎯 Цель проекта:** 
---
Создать полностью рабочий кластер Kubernetes на Ubuntu 22.04, включающий:

1 Master Node

2 Worker Nodes

Flannel — как сетевой плагин для связи подов (CNI)

Kubernetes Dashboard — для визуального управления кластером

Также мы проверим:

автоматическое распределение нагрузки между нодами,

восстановление подов при отказе одной из нод,

## 🧩 1. Подготовка системы

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl gpg
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

---

## 🧩 2. Настройка сетевых параметров

```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
```

---

## 🧩 3. Установка containerd

```bash
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

## 🧩 4. Установка kubeadm, kubelet, kubectl

```bash
sudo mkdir -p /etc/apt/keyrings
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## 🧩 5. Инициализация мастер-ноды

```bash
sudo kubeadm init --apiserver-advertise-address=172.16.18.196 --pod-network-cidr=10.244.0.0/16
```

После инициализации появится сообщение:

> Your Kubernetes control-plane has initialized successfully!

---

## 🧩 6. Настройка kubectl

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl get nodes
```

---

## 🧩 7. Установка Flannel

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
sudo mkdir -p /opt/cni/bin
curl -L -o cni-plugins.tgz https://github.com/containernetworking/plugins/releases/download/v1.4.0/cni-plugins-linux-amd64-v1.4.0.tgz
sudo tar -C /opt/cni/bin -xzvf cni-plugins.tgz
sudo systemctl restart kubelet
```

Проверяем:
```bash
kubectl get pods -n kube-system
kubectl get nodes
```

Через 1–2 минуты нода должна быть `Ready`.

---

## ⚙️ УСТАНОВКА WORKER-НОДЫ

**IP воркера:** 172.16.18.164 (или другой)

Повтори шаги 1–4 (подготовка, сеть, containerd, kubeadm).  
Затем создай конфиг Flannel:

```bash
sudo tee /etc/cni/net.d/10-flannel.conflist > /dev/null <<'EOF'
{
  "name": "cbr0",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "flannel",
      "delegate": {
        "hairpinMode": true,
        "isDefaultGateway": true
      }
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    }
  ]
}
EOF

sudo systemctl restart containerd
sudo systemctl restart kubelet
```

Присоедини воркер:

```bash
sudo kubeadm join 172.16.18.196:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

Проверка на мастере:

```bash
kubectl get nodes
```

---

## 🧩 УСТАНОВКА KUBERNETES DASHBOARD

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

Создаём пользователя администратора:

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

Получаем токен:

```bash
kubectl -n kubernetes-dashboard create token admin-user
```

Изменяем тип сервиса:

```bash
kubectl -n kubernetes-dashboard edit service kubernetes-dashboard
```

Меняем строку:
```
type: ClusterIP
```
на
```
type: NodePort
```

Проверяем порт:
```bash
kubectl -n kubernetes-dashboard get svc
```

Открываем Dashboard:
```
https://<IP_МАСТЕРА>:<NodePort>
```
Вводим токен — и готово 🎉

---

## 🧩 Пример деплоя приложения

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: activar-deployment
  labels:
    app: activar
spec:
  replicas: 2
  selector:
    matchLabels:
      app: activar
  template:
    metadata:
      labels:
        app: activar
    spec:
      containers:
      - name: activar
        image: jacobvell/activar:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: activar-service
spec:
  selector:
    app: activar
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: NodePort
```

---

## 🧹 Удаление воркер-ноды

На мастере:
```bash
kubectl drain worker-node1 --delete-emptydir-data --force --ignore-daemonsets
kubectl delete node worker-node1
```

На воркере:
```bash
sudo kubeadm reset -f
sudo ip link delete cni0
sudo ip link delete flannel.1
sudo rm -rf /etc/cni/net.d /var/lib/cni/ /var/lib/kubelet/* ~/.kube
sudo iptables -F
sudo iptables -t nat -F
sudo iptables -X
```

---

🧠 **Готово!**  
Теперь у тебя есть полностью рабочий кластер Kubernetes с Dashboard и деплоем своего приложения.
