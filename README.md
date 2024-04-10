# llama-kubernetes

Даже с маленьким ~~IQ~~ RAM можно ~~сделать BBQ~~ запустить большую языковую модель...

Недавно я опробовал GitHub Copilot - AI-ассистент для написания кода. В качестве подопытного проекта я использовал https://github.com/ruslanbay/moex Исходя из этого эксперимента, я сделал вывод что на данном этапе ассистент не напишет приложение с нуля, но если основная логика реализована, то он вполне пригоден для рефакторинга. Тем не менее это был интересный опыт и в большинстве случаев ассистент справлялся с поставленной задачей.

Чат боты требовательны к аппартаным ресурсам и потребляют много электроэнергии в процессе работы. В интернете я нашёл несколько моделей специально обученных для написания кода. Например, модель Phi2 от Microsoft включает всего два миллиарда параметров, но её [навыки написания кода не уступают большим моделям](https://www.microsoft.com/en-us/research/blog/phi-2-the-surprising-power-of-small-language-models/).
![image](https://github.com/ruslanbay/llama-kubernetes/blob/test/images/coding_benchmark.png)

В интернете доступны модели, которые можно запустить даже на мобильном устройстве. Я написал этот пост, чтобы продемонстрировать что для запуска языковой модели не требуется компьютер с новейшим GPU и терабайтами оперативной памяти. Замечу, я не ставил перед собой задачу обучения языковой модели - для этого всё же требуется мощная видеокарта и значительные объёмы памяти.

 1. Выбор операционной системы
 2. Активируем ZRAM
 3. Развертывание Kubernetes (k0s)
 4. 

## Выбор операционной системы

Я часто переезжаю, поэтому единственный компьютер, которым я пользуюсь на данный момент - планшет Surface Pro 7 с процессором Core i5-1035G4 и 8Гбайт оперативной и 128 Гбайт постоянной памяти. Core i5-1035G4 - один из первых потребительских процессоров с поддержкой инструкций AVX-512. В теории, инструкции AVX-512 должны обеспечить лучшую производительность в задачах машинного обучения. Чтобы восползьоваться этим преимуществом я установил на планшет операционную систему [ClearLinux](https://www.clearlinux.org/documentation/clear-linux/get-started.html) - дистрибутив от Intel, который содержит бинарные пакеты оптимизированные для архитектуры [x86-64-v4](https://en.wikipedia.org/wiki/X86-64#Microarchitecture_levels).


## Активируем Zram

Как я уже упомянул, запуск больших языковых моделей требует большого объёма оперативной памяти. Чтобы посчитать примерный объём памяти можно воспользоваться [формулой](https://www.substratus.ai/blog/calculating-gpu-memory-for-llm):

M = P∗4B∗1.2 / (32/Q), где

|Symbol |Description|
|-------|-----------|
|M	|GPU memory expressed in Gigabyte|
|P	|The amount of parameters in the model. E.g. a 7B model has 7 billion parameters.|
|B	|4 bytes, expressing the bytes used for each parameter|
|32	|There are 32 bits in 4 bytes|
|Q	|The amount of bits that should be used for loading the model. E.g. 16 bits, 8 bits or 4 bits.|
|1.2	|Represents a 20% overhead of loading additional things in GPU memory.|

В теории, 8 Гбайт RAM на моём планешете должно быть достаточно для запуска модели с 2B параметрами. Чтобы перестраховаться я активировал Zram. В отличии от файла подкачки, который производит запись/считывание данных с диска, Zram при нехватке RAM выполняет сжатие данных "на лету" прямо в оперативной памяти. В зависимости от алгоритма сжатия и типа данных, [Zram позволяет записать в оперативную память данных в несколько раз больше физического объёма RAM](https://linuxreviews.org/Zram). Zstd и lzo-rle - наиболее оптимальные алгоритмы по соотношению степени сжания к скорости выполнения этого самого сжатия.

Отключим swapfile, который активирован по умолчанию в ClearLinux:
```bash
sudo systemctl mask $(sed -n -e 's#^/var/\([0-9a-z]*\).*#var-\1.swap#p' /proc/swaps) 2>/dev/null
sudo swapoff -a
```

Активируем Zram:

Найдём доступный объём оперативной памяти:
```bash
free -b
```

Используем найденное значение в качестве значения аргумента size для следующей команды:
```bash
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

Выполним следующую команду, чтобы Zram активировался автоматически после старта системы:
```bash
sudo systemctl enable --now zram.service
```


## Развертывание Kubernetes (k0s)

Чтобы иметь простор для экспериментов, я воспользовался контейнерезацией. К тому же, это был хороший повод опробовать k0s - легковесный дистрибутив Kubernetes "всё-в-одном".

Активируем IP forwarding <sup><a href="https://www.clearlinux.org/clear-linux-documentation/tutorials/kubernetes.html#set-up-kubernetes-manually">[источник]</a></sup>:

```bash
sudo mkdir -p /etc/sysctl.d/

sudo tee /etc/sysctl.d/99-kubernetes-cri.conf > /dev/null <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
```

Добавим hostname в /etc/hosts file:
```bash
echo "127.0.0.1 localhost `hostname`" | sudo tee --append /etc/hosts
```

В отличии от полноценного Kubernetes, [k0s поставляется с предустановленным CNI](https://docs.k0sproject.io/stable/networking/), нет необходимости устанавливать его вручную.

k0s по умолчанию использует containerd. Чтобы добавить containerd установим [containers-basic](https://www.clearlinux.org/software/bundle/containers-basic.html):
```bash
sudo swupd bundle-add containers-basic
sudo systemctl enable --now containerd.service
```

Из недостатков ClearLinux - дополнительное программное обеспечение поставляется в виде bundle-ов, которые часто содержат пакеты, которые не используются и просто занимают место на диске. Помимо containerd, containers-basic так же содержит docker и cri-o. Чтобы избежать путаницы, отключим неиспользуемые сервисы:
```bash
sudo systemctl disable crio.service
sudo systemctl disable docker.socket docker.service
```

Теперь, собственно, устанавливаем сам k0s<a href=https://docs.k0sproject.io/stable/install/><sup>источник</sup></a>:
```bash
curl -sSLf https://get.k0s.sh | sudo sh

Отключаем встроенный metrics apiservice, так как мы будем использовать Prometheus
sudo k0s install controller --single --disable-components metrics-server

sudo k0s start

Этот шаг нужен, чтобы избежать некоторых ошибок и повторного скачивания образов каждый раз после выполнения команды `k0s reset`
sudo ln -s /var/run/containerd/containerd.sock /run/k0s/containerd.sock
```



https://huggingface.co/TheBloke/phi-2-GGUF#provided-files




Для сравнения Gemma потребовалось более 20 минут на то чтобы сгененрировать ответ.



1. Установка и настройка операционной системы

https://www.clearlinux.org/clear-linux-documentation/tutorials/kubernetes.html#set-up-kubernetes-manually




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
