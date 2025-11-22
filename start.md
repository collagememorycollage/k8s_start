1) отключаем файл подкачки так как kubelet не работает с включенным файлом подкачки
sudo swapoff -a

2) также отключаем данный файл после перезагрузки
sudo sed -i 's/^\(.*swap.*\)$/#\1/' /etc/fstab

3) обновляем пакеты
sudo apt get update -y

4) добавляем репы на keying
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o \
/etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | \
sudo tee /etc/apt/sources.list.d/kubernetes.list

5) повторное обновление системы
sudo apt-get update -y

6) устанавливаем пакеты самого кубера
kubelet будет как раз запускать  наши контейнеры через containerd
kubeadm для того, чтобы наш кластер развернуть
kubectl для того, чтобы взаимодействовать с кластером
sudo apt-get install -y kubelet kubeadm kubectl containerd

7) sudo apt-mark hold kubelet kubeadm kubectl

8) переходим на root
sudo su

9) Далее активируем модуль ядра
modprobe br_netfilter
modprobe overlay

10) для того, чтобы модули ядра активировались сами, необходимо прописать

echo -e "br_netfilter\noverlay" | sudo tee -a /etc/modules

11) делаем проброс портов на самих машинах
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf

12) устанавливаем бридж
echo "net.bridge.bridge-nf-call-iptables=1" >> /etc/sysctl.conf

13) после проверяем что правила для сети применились

14) создаем папку для containerd
sudo mkdir /etc/containerd

15) далее настроим конфиг для нашего containerd
sudo nano /etc/containerd/config.toml

16) вставляем наш конфиг и перезапускаем службу
version = 2
[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
   [plugins."io.containerd.grpc.v1.cri".containerd]
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          runtime_type = "io.containerd.runc.v2"   
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true

systemctl restart containerd

17) получаем ip мастер ноды, чтобы иницииализировать кластер
ip a

18) разворачиваем нашу мастер ноду
где 10.128.0.28 это ip мастер ноды, а 10.244.0.0/16 ip подсети для нашего приложения
sudo kubeadm init \
  --apiserver-advertise-address=10.128.0.28 \
  --pod-network-cidr 10.244.0.0/16
   
также можно добавить флаг, чтобы добавить внешний ip адрес --apiserver-extra-sans=158.100.2.1

19) устанавливаем дефолнтые настройки kubeconfig
mkdir -p $HOME/.kube (создаем на каждой ноде)
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

после копируем нашу сгенерированную конфигурацию с мастер ноды на остальные

# install cni flannel
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

Только на worker Nodes
add worker nodes
kubeadm token generate - на мастере генерим ключ (1)
kubeadm token create <generated-token> --print-join-command --ttl=0 - на мастере вставляем ключ (2)

после получим на мастере команду и раскидываем ее по воркер нодам
sudo kubeadm join 10.128.0.28:6443 --token zvxm7y.z61zq4rzaq3rtipk \
        --discovery-token-ca-cert-hash sha256:9b650e50a7a5b6261746684d033a7d6483ea5b84db8932cb70563b35f91080f7
kubectl get nodes - должно быть все Ready
Если что-то не Ready, попробовать перезапустить

sudo systemctl restart containerd
sudo systemctl restart kubelet



