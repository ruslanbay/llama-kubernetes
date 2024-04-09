# llama-kubernetes

Даже с маленьким ~~IQ~~ RAM можно ~~сделать BBQ~~ запустить большую языковую модель...

Недавно я опробовал GitHub Copilot - AI-ассистент для написания кода. В качестве подопытного проекта я использовал https://github.com/ruslanbay/moex В целом мои ожидания оправдались. Ассистент не напишет приложение с нуля, но если основная логика реализована, то вполне пригоден для рефакторинга. Тем не менее это был интересный опыт и в большинстве случаев ассистент справлялся с поставленной задачей.

Я задумался как реализована серверная часть Copilot. Чат боты требовательны к аппартаным ресурсам и потребляют много электроэнергии в процессе работы. Вероятно Microsoft использует для ассистента языковую модель специально обученную для написания кода. Я не знаю какую именно модель использует Microsoft для Copilot, но не 

Для сравнения Gemma потребовалось более 20 минут на то чтобы сгененрировать ответ.



1. Установка и настройка операционной системы

https://www.clearlinux.org/clear-linux-documentation/tutorials/kubernetes.html#set-up-kubernetes-manually

```bash
free -b

cat <<EOF | sudo tee /etc/systemd/system/zram.service
[Unit]
Description=Enable ZRAM
After=network.target

[Service]
Type=idle
ExecStart=/bin/bash -c 'modprobe zram && zramctl /dev/zram0 --algorithm lzo-rle --size 7892946944 && mkswap -U clear /dev/zram0 && swapon --priority 100 /dev/zram0'

[Install]
WantedBy=default.target
EOF
```


```bash
sudo mkdir -p /etc/sysctl.d/

sudo tee /etc/sysctl.d/99-kubernetes-cri.conf > /dev/null <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

2. Установка containerd

`sudo swupd bundle-add containers-basic`



3. Установка k0s

```bash
sudo systemctl disable crio.service
sudo systemctl disable docker.socket docker.service
sudo systemctl enable --now containerd.service
```

curl -sSLf https://get.k0s.sh | sudo sh

Отключаем встроенный metrics apiservice, так как мы будем использовать Prometheus
```bash
sudo k0s install controller --single --disable-components metrics-server
```

```bash
sudo k0s start
```

Этот шаг нужен, чтобы избежать некоторых ошибок и повторного скачивания образов каждый раз после выполнения команды `k0s reset`
sudo ln -s /var/run/containerd/containerd.sock /run/k0s/containerd.sock


4. В k0s нет возможности изменить конфигурацию существуюего инстанса. Вместо этого нужно выполнить установку повторно:
sudo k0s install controller --single --disable-components metrics-server --force
sudo systemctl daemon-reload
sudo k0s start



4. Развёртывание мониторинга

https://github.com/prometheus-operator/kube-prometheus?tab=readme-ov-file#quickstart


cd /home/admin/workdir/
git clone https://github.com/prometheus-operator/kube-prometheus.git kube-prometheus
sudo k0s kubectl apply --server-side -f manifests/setup
sudo k0s kubectl wait \
	--for condition=Established \
	--all CustomResourceDefinition \
	--namespace=monitoring
sudo k0s kubectl apply -f manifests/

sudo k0s kubectl get pod -monitoring -w


5. Развёртывание llama-server



6. Метрики llama-server в Grafana



7. Удаление

Удаляем llama-server
sudo k0s kubectl delete statefulset llama-server

Удаляем мониторинг
https://github.com/prometheus-operator/kube-prometheus?tab=readme-ov-file#quickstart
cd /home/admin/workdir/kube-prometheus
sudo k0s kubectl delete --ignore-not-found=true -f manifests/ -f manifests/setup

Удаляем k0s
https://docs.k0sproject.io/v1.29.2+k0s.0/reinstall-k0sctl/#reinstall-a-node
sudo k0s kubectl drain --ignore-daemonsets --delete-emptydir-data ${HOSTNAME}
sudo k0s stop
sudo k0s reset
sudo systemctl reboot
