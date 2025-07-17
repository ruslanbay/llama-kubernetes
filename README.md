# Развёртываем Kubernetes с большой языковой моделью на компьютере c 8 Гбайт RAM без дискретной видеокарты

![image](/images/futurama.png)

## Настройка хост системы

На моём компютере установлена Fedora 42. 

```shell
sudo dnf install -y --setopt=install_weak_deps=FALSE \
  qemu-system-x86_64 \
  qemu-img
```

## Создание виртуальной машины (гостевая система) <sup>[docs](https://www.qemu.org/docs/master/system/qemu-manpage.html)</sup>

```shell
VM_NAME=alpine-k0s
VM_DIR="${HOME}/VMs"
CPU_SOCKETS=1
CPU_CORES_PER_SOCKET=4
CPU_THREADS_PER_CORE=2
RAM_SIZE=6G
DISK_SIZE=15G
DISK_PATH="${VM_DIR}/${VM_NAME}.qcow2"

mkdir -p "${VM_DIR}"

ISO_URL="https://dl-cdn.alpinelinux.org/alpine/v3.22/releases/x86_64/alpine-virt-3.22.0-x86_64.iso"
ISO_PATH="${VM_DIR}/alpine-virt-3.22.0-x86_64.iso"

[ ! -f "$ISO_PATH" ] && curl -L -o "$ISO_PATH" "$ISO_URL"

[ ! -f "$DISK_PATH" ] && qemu-img create -f qcow2 "$DISK_PATH" $DISK_SIZE

qemu-system-x86_64 \
  -name "$VM_NAME" \
  -machine type=q35,accel=kvm \
  -enable-kvm \
  -cpu host \
  -smp sockets=$CPU_SOCKETS,cores=$CPU_CORES_PER_SOCKET,threads=$CPU_THREADS_PER_CORE \
  -m $RAM_SIZE \
  -drive file="$DISK_PATH",if=virtio,format=qcow2 \
  -cdrom "$ISO_PATH" \
  -netdev user,id=net0,hostfwd=tcp::30088-:30088,hostfwd=tcp::30434-:30434 \
  -device virtio-net-pci,netdev=net0 \
  -nographic
```

## Установка Alpine Linux <sup>[docs](https://wiki.alpinelinux.org/wiki/Alpine_setup_scripts#setup-alpine)</sup>

Чтобы установить Alpine Linux выполните команду `setup-alpine`. Этот скрипт проведёт вас через интерактивную настройку клавиатуры, имени хоста, сети, пароля root, часового пояса и других параметров.

Вы можете автоматизировать процесс установки, создав файл ответов, который содержит предопределённые ответы на все запросы: `setup-alpine -f answerfile.txt`. Пример файла ответов приведён ниже.

```shell
# Example answer file for setup-alpine script
# If you don't want to use a certain option, then comment it out

# Use US layout with US variant
# KEYMAPOPTS="us us"
KEYMAPOPTS=none

# Set hostname to 'alpine'
HOSTNAMEOPTS=alpine

# Set device manager to mdev
DEVDOPTS=mdev

# Contents of /etc/network/interfaces
INTERFACESOPTS="auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp
hostname alpine-test
"

# Search domain of example.com, Google public nameserver
# DNSOPTS="-d example.com 8.8.8.8"

# Set timezone to UTC
#TIMEZONEOPTS="UTC"
TIMEZONEOPTS=none

# set http/ftp proxy
#PROXYOPTS="http://webproxy:8080"
PROXYOPTS=none

# Add first mirror (CDN)
APKREPOSOPTS="-1"

# Create admin user
USEROPTS="-a -u -g audio,input,video,netdev juser"
#USERSSHKEY="ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOIiHcbg/7ytfLFHUNLRgEAubFz/13SwXBOM/05GNZe4 juser@example.com"
#USERSSHKEY="https://example.com/juser.keys"

# Install Openssh
SSHDOPTS=openssh
#ROOTSSHKEY="ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOIiHcbg/7ytfLFHUNLRgEAubFz/13SwXBOM/05GNZe4 juser@example.com"
#ROOTSSHKEY="https://example.com/juser.keys"

# Use openntpd
# NTPOPTS="openntpd"
NTPOPTS=none

# Use /dev/sda as a sys disk
# DISKOPTS="-m sys /dev/sda"
DISKOPTS=none

# Setup storage with label APKOVL for config storage
#LBUOPTS="LABEL=APKOVL"
LBUOPTS=none

#APKCACHEOPTS="/media/LABEL=APKOVL/cache"
APKCACHEOPTS=none
```

После завершения установки Alpine Linux перезагрузите систему командой `reboot`. Авторизуйтесь в системе с учётной записью `root` и паролем, который вы указали во время установки.

## Добавление пакетов

```shell
apk add \
    bash \
    curl \
    envsubst \
    git \
    zram-init
```

## Активируем Zram <sup>[docs](https://wiki.alpinelinux.org/wiki/Zram)</sup>

Модуль Zram создаёт в оперативной памяти блочное устройство, которое ускоряет операции ввода-вывода и обеспечивает экономию памяти за счёт сжатия.

### Отключение раздела подкачки

Для начала необходимо отключить раздел подкачки, который Alpine Linux создаёт по умолчанию.

```shell
swapoff -a
sed -i '/swap/ s/^/#/' /etc/fstab
```

Перезагрузите систему командой `reboot`, чтобы убедиться, что раздел подкачки полностью отключён.

### Настройка и запуск сервиса Zram

Ниже приведён пример конфигурационного файла `/etc/conf.d/zram-init`.

```shell
load_on_start=yes
unload_on_stop=yes
num_devices=1

type0=swap
flag0= # The default "16383" is fine for us

size0=`LC_ALL=C free -m | awk '/^Mem:/{print int($2/1)}'`
mlim0= # no hard memory limit
back0= # no backup device
icmp0= # no incompressible page writing to backup device
idle0= # no idle page writing to backup device
wlim0= # no writeback_limit for idle page writing for backup device
notr0= # keep the default on linux-3.15 or newer
maxs0=1 # maximum number of parallel processes for this device
algo0=zstd # zstd (since linux-4.18), lz4 (since linux-3.15), or lzo.
           # Size: zstd (best) > lzo > lz4. Speed: lz4 (best) > zstd > lzo
labl0=zram_swap # the label name
uuid0= # Do not force UUID
args0= # we could e.g. have set args0="-L 'zram_swap'" instead of using labl0
```

Запустим сервис Zram. В системе будет создано блочное устройство `/dev/zram0`, которое будет использоваться в качестве раздела подкачки. Размер этого устройства будет равен объёму оперативной памяти, доступной в системе.

```shell
rc-service zram-init start
```

Для автоматического запуска сервиса выполните следующую команду:
```shell
rc-update add zram-init
```

## Развёртываем k0s <sup>[docs](https://docs.k0sproject.io/stable/install/)</sup>

K0s — это дистрибутив Kubernetes с открытым исходным кодом, который включает в себя все необходимые функции для создания кластера Kubernetes и упакован в один исполняемый файл для удобства использования.

```shell
curl -sSf https://get.k0s.sh | sh
k0s install controller --single --disable-components metrics-server
k0s start
k0s status

echo "alias k='k0s kubectl'" >> ~/.profile

k get nodes
```

Дождитесь пока все поды перейдут в статус `Running` для этого выполните команду `k0s kubectl get pod -A -w` или `k get pod -A -w`.

## Склонируем репозиторий

```shell
cd /root

git clone https://github.com/ruslanbay/llama-kubernetes
```

## Запустим контейнер с Ollama <sup>[docs](https://github.com/ollama/ollama/blob/main/docs/docker.md)</sup>

[Ollama](https://github.com/ollama/ollama) - это инструмент с открытым исходным кодом для запуска и управления языковыми моделями. В качестве бекенда используется [llama.cpp](https://github.com/ggml-org/llama.cpp).

```shell
k0s kubectl create ns ollama

mkdir -p /root/.ollama/models

export MODELS_PATH=/root/.ollama/models

envsubst < /root/llama-kubernetes/ollama.yaml | k0s kubectl -n ollama apply -f -
```

### Проверим статус пода и сервиса
```shell
localhost:~# k -n ollama get all 
NAME                          READY   STATUS    RESTARTS   AGE
pod/ollama-758f8b79f9-7f6jq   1/1     Running   0          55m

NAME                     TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
service/ollama-service   NodePort   10.101.23.190   <none>        11434:30434/TCP   55m

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ollama   1/1     1            1           55m

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/ollama-758f8b79f9   1         1         1       55m
```

### Пример использования языковой модели [qwen3:0.6b](https://ollama.com/library/qwen3)

```shell
k0s kubectl exec -n ollama ollama-758f8b79f9-7f6jq -it -- ollama run qwen3:0.6b --think --verbose --nowordwrap "what are you?"
```

Кроме того, можно воспользоваться [Rest API](https://github.com/ollama/ollama/blob/main/docs/api.md). Пример запроса:

```shell
curl -s http://localhost:30434/api/tags | jq
{
  "models": [
    {
      "name": "qwen3:0.6b",
      "model": "qwen3:0.6b",
      "modified_at": "2025-07-14T10:45:57.174Z",
      "size": 522653767,
      "digest": "7df6b6e09427a769808717c0a93cadc4ae99ed4eb8bf5ca557c90846becea435",
      "details": {
        "parent_model": "",
        "format": "gguf",
        "family": "qwen3",
        "families": [
          "qwen3"
        ],
        "parameter_size": "751.63M",
        "quantization_level": "Q4_K_M"
      }
    }
  ]
}
```

## Запуск контейнера vscode

Развернём контейнер с Visual Studio Code. Vscode будет использовать Ollama в качестве бэкенда для работы с языковыми моделями. Значение переменной VSCODE_CONNECTION_TOKEN будет использоваться для доступа к веб-интерфейсу vscode. В примере ниже используется токен `my-token`, но вы можете указать собственное значение:

```shell
k0s kubectl create ns vscode

mkdir -p /root/.vscode-server/extensions
mkdir -p /root/.vscode-server/data/Machine

export VSCODE_CONNECTION_TOKEN="my-token"
export VSCODE_DATA_PATH="/root/.vscode-server"

envsubst < /root/llama-kubernetes/vscode.yaml | k0s kubectl -n vscode apply -f -
```

Убедимся, что под и сервис запущены:

```shell
localhost:~# k get all -n vscode
NAME                                 READY   STATUS    RESTARTS   AGE
pod/vscode-server-7499694fc7-cqqdn   1/1     Running   0          95m

NAME                    TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
service/vscode-server   NodePort   10.97.2.188   <none>        8088:30088/TCP   95m

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/vscode-server   1/1     1            1           95m

NAME                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/vscode-server-7499694fc7   1         1         1       95m
```

Чтобы получить доступ к vscode откройте браузер в хост системе и перейдите по адресу `http://localhost:30088/?tkn=my-token`. Вместо `my-token` используйте значение переменной `VSCODE_CONNECTION_TOKEN`, которое вы указали на предыдущем шаге.

## Интеграция VS Code с Ollama

### AI Toolkit for Visual Studio Code

В VS Code установите расширение [AI Toolkit for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=ms-windows-ai-studio.windows-ai-studio). Затем в каталоге языковых моделей выберете `Your Ollama Model` > `Add Your Own`.

![image](/images/ai_toolkit_1.png)

Сверху появится выподающий список - выберете `Provide custom Ollama endpoint`.

![image](/images/ai_toolkit_2.png)

В появившемся поле введите `http://ollama-service.ollama:11434`. Здесь ollama-service - имя сервиса, ollama - название namespace. Укажите собственные значения, если вы редактировали эти параметры.

![image](/images/ai_toolkit_3.png)

Снова нажмите `Your Ollama Model` > `Add Your Own` и выберете `Select models from Ollama library`. В выпадающем списке выберете языковую модель, в этом примере я использую `qwen3:0.6b`. Затем перейдите в раздел `Playground` - здесь вы можете взаимодействовать с языковой моделью и задать её параметры.

![image](/images/ai_toolkit_4.png)

### GitHub Copilot Chat

В VS Code установите расширение [GitHub Copilot Chat](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot-chat) и авторизуйтесь с помощью учётной записи GitHub. Далее откройте настройки и укажите `http://ollama-service.ollama:11434` в качестве Ollama Endpoint. Здесь ollama-service - имя сервиса, ollama - название namespace. Укажите собственные значения, если вы редактировали эти параметры.

![image](/images/copilot_chat.png)

В окне чата разверните список языковых моделей и нажмите `Manage Models...`

![image](/images/copilot_chat_1.png)

Выберете Ollama из списка провайдеров:

![image](/images/copilot_chat_2.png)

Выберете языковую модель:

![image](/images/copilot_chat_3.png)

Готово! Теперь вы можете использовать в VS Code локальную языковую модель:

![image](/images/copilot_chat_4.png)