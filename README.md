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

Найдём доступный объём оперативной памяти:
```bash
free -b
```

Зададим найденное значение для аргумента size команды zramctl:
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

Активируем IP forwarding <sup><a style="text-decoration:none" href="https://www.clearlinux.org/clear-linux-documentation/tutorials/kubernetes.html#set-up-kubernetes-manually">[источник]</a></sup> :

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

> [!NOTE]
> По идее, установка containerd тоже не требуется, но мне пришлось установить его отдельно, чтобы обойти некоторые ошибки

k0s по умолчанию использует containerd. Чтобы добавить containerd установим [containers-basic](https://www.clearlinux.org/software/bundle/containers-basic.html):
```bash
sudo swupd bundle-add containers-basic
sudo systemctl enable --now containerd.service
```

Из недостатков ClearLinux - дополнительное программное обеспечение поставляется в виде bundle-ов, которые зачастую содержат лишние зависимости пакеты и занимают много дискового пространства. Помимо containerd, containers-basic так же содержит docker и cri-o. Чтобы избежать путаницы, отключим неиспользуемые сервисы:
```bash
sudo systemctl disable crio.service
sudo systemctl disable docker.socket docker.service
```

Теперь, собственно, устанавливаем сам k0s <a style="text-decoration:none" href="https://docs.k0sproject.io/stable/install/"><sup>[источник]</sup></a> :
```bash
curl -sSLf https://get.k0s.sh | sudo sh
```

Отключаем встроенный metrics apiservice, так как мы будем использовать Prometheus
```bash
sudo k0s install controller --single --disable-components metrics-server
sudo k0s start
```
Этот шаг нужен, чтобы избежать некоторых ошибок и повторного скачивания образов каждый раз после выполнения команды `k0s reset`
```bash
sudo ln -s /var/run/containerd/containerd.sock /run/k0s/containerd.sock
```

Развертывание Kubernetes завершено. Чтобы проверить статус ноды и запущенных контейнеров выполним следующие команды:
```bash
sudo k0s status
sudo k0s kubectl get pods -A -w
```

> [!NOTE]
> В k0s нет возможности изменить конфигурацию существуюего инстанса. Вместо этого нужно выполнить установку повторно:
> ```bash
> sudo k0s install controller --single --disable-components metrics-server --force
> sudo systemctl daemon-reload
> sudo k0s start
> sudo ln -s /var/run/containerd/containerd.sock /run/k0s/containerd.sock
> ```

## Добавляем мониторинг <a style="text-decoration:none" href="https://github.com/prometheus-operator/kube-prometheus?tab=readme-ov-file#quickstart"><sup>[источник]</sup></a>

Создадим в домашнем каталоге директорию `workdir`:
```bash
mkdir ~/workdir
```

В директории `workdir` разместим файлы, связанные с проектом:
```bash
git clone https://github.com/prometheus-operator/kube-prometheus.git ~/workdir/kube-prometheus
cd ~/workdir/kube-prometheus
sudo k0s kubectl apply --server-side -f manifests/setup
sudo k0s kubectl wait \
	--for condition=Established \
	--all CustomResourceDefinition \
	--namespace=monitoring
sudo k0s kubectl apply -f manifests/
```

Дождёмся когда все контейнеры перейдут в статус `Running`
```bash
sudo k0s kubectl get pod -monitoring -w
```

## Компилируем llama server
Если в вашем случае встроенная графика Intel содержит более 80 вычислительных блоков (Execution Unit, EU), то можете выполнить установку по инструкции [llama.cpp for SYCL](https://github.com/ggerganov/llama.cpp/blob/master/README-sycl.md). Я использую [процессор i5-1035g4, который имеет интегрированную графику с 48 EU](https://en.wikichip.org/wiki/intel/core_i5/i5-1035g4#Graphics) - этого недостаточно для комфортного использования языковой модели. Поэтому я воспользуюсь [intel-onemkl](https://github.com/ggerganov/llama.cpp/tree/master#intel-onemkl) для компиляции llama.cpp

Сколонируем репозиторий llama.cpp в каталог `~/workdir/llama.cpp`:
```bash
git clone -b b2420 https://github.com/ggerganov/llama.cpp.git ~/workdir/llama.cpp
```

Запустим контейнер:

> [!WARNING]
> Потребуется примерно 20Гбайт дискового пространства.

```bash
sudo k0s ctr image pull docker.io/intel/oneapi-basekit:2024.1.0-devel-ubuntu22.04
sudo k0s ctr run --rm --mount type=bind,src=/home/admin/workdir/llama.cpp,dst=/mnt,options=rbind:rw --cwd /mnt docker.io/intel/oneapi-basekit:2024.1.0-devel-ubuntu22.04 test-container-name
```

В контейнере выполним следующие команды:
```bash
mkdir build
cd build
cmake .. -DLLAMA_BLAS=ON -DLLAMA_BLAS_VENDOR=Intel10_64lp -DCMAKE_C_COMPILER=icx -DCMAKE_CXX_COMPILER=icpx -DLLAMA_NATIVE=ON
cmake --build . --config Release
```

Контейнер `oneapi-basekit` занимает примерно 20Гбайт дискового пространства. После завершения компиляции чтобы высвободить место на диске удалим этот контейнер, а для запуска llama server будем использовать образ `oneapi-runtime`.


## Запускаем llama server в виде statefulset

Склонируем репозиторий https://github.com/ruslanbay/llama-kubernetes в каталог `~/workdir/llama-kubernetes`:
```bash
git clone https://github.com/ruslanbay/llama-kubernetes.git ~/workdir/llama-kubernetes
cd ~/workdir/llama-kubernetes
```

Для примера воспользуемся моделью [phi-2.Q5_K_M](https://huggingface.co/TheBloke/phi-2-GGUF). Загрузим её и поместим в каталог `~\workdir\llama.cpp\models\`
```bash
curl https://huggingface.co/TheBloke/phi-2-GGUF/resolve/main/phi-2.Q5_K_M.gguf?download=true -o ~\workdir\llama.cpp\models\phi-2.Q5_K_M.gguf
```

Если запустить llama server с опцией `--metrics`, то будет добавлен endpoint `/metrics` со следующими метриками <a style="text-decoration:none" href="https://github.com/ggerganov/llama.cpp/blob/master/examples/server/README.md#result-json-2"><sup>[источник]</sup></a> :
|Metric					|Description					|
|---------------------------------------|-----------------------------------------------|
|llamacpp:prompt_tokens_total		|Number of prompt tokens processed.		|
|llamacpp:tokens_predicted_total	|Number of generation tokens processed.		|
|llamacpp:prompt_tokens_seconds		|Average prompt throughput in tokens/s.		|
|llamacpp:predicted_tokens_seconds	|Average generation throughput in tokens/s.	|
|llamacpp:kv_cache_usage_ratio		|KV-cache usage. 1 means 100 percent usage.	|
|llamacpp:kv_cache_tokens		|KV-cache tokens.				|
|llamacpp:requests_processing		|Number of requests processing.			|
|llamacpp:requests_deferred		|Number of requests deferred.			|

Добавим servicemonitor чтобы Prometheus мог считать эти метрики:
```bash
sudo k0s kubectl apply -f servicemonitor/llama-metrics.yaml
```

Теперь развернём непосредственно сам llama server:
```bash
sudo k0s kubectl apply -f service/llama-server.yaml
sudo k0s kubectl apply -f storage/llama-storage-class.yaml
sudo k0s kubectl apply -f pv/llama.yaml
sudo k0s kubectl apply -f pvc/llama.yaml
sudo k0s kubectl apply -f statefulset/llama-server.yaml
```


## Пример использования модели

Вывести список всех сервисов:
```bash
sudo k0s kubectl get svc -A
```

Нас интересуют сервисы llama-server и grafana:

Попросим модель сегенировать поздравление со свадьбой. Как видим, модель справилась с задачей за 3 секунды:
```bash
curl -X POST http://10.244.0.143:8080/completion \
     --header "Content-Type: application/json" \
     --data '{"prompt": "Generate a wedding greeting for my friend named XY and his wife XX.","n_predict": 128}' | json_pp

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100  1604  100  1498  100   106    386     27  0:00:03  0:00:03 --:--:--   413

{
   "content" : "\nInput: XY and XX\nOutput: I'm so excited to announce the marriage of XX and XY! May your love story be as beautiful as the one we've witnessed today. Wishing you a lifetime of happiness and togetherness!\n",
   "generation_settings" : {
      "dynatemp_exponent" : 1,
      "dynatemp_range" : 0,
      "frequency_penalty" : 0,
      "grammar" : "",
      "ignore_eos" : false,
      "logit_bias" : [],
      "min_keep" : 0,
      "min_p" : 0.0500000007450581,
      "mirostat" : 0,
      "mirostat_eta" : 0.100000001490116,
      "mirostat_tau" : 5,
      "model" : "/mnt/models/phi-2.Q5_K_M.gguf",
      "n_ctx" : 512,
      "n_keep" : 0,
      "n_predict" : -1,
      "n_probs" : 0,
      "penalize_nl" : true,
      "penalty_prompt_tokens" : [],
      "presence_penalty" : 0,
      "repeat_last_n" : 64,
      "repeat_penalty" : 1.10000002384186,
      "samplers" : [
         "top_k",
         "tfs_z",
         "typical_p",
         "top_p",
         "min_p",
         "temperature"
      ],
      "seed" : 4294967295,
      "stop" : [],
      "stream" : false,
      "temperature" : 0.800000011920929,
      "tfs_z" : 1,
      "top_k" : 40,
      "top_p" : 0.949999988079071,
      "typical_p" : 1,
      "use_penalty_prompt_tokens" : false
   },
   "id_slot" : 0,
   "model" : "/mnt/models/phi-2.Q5_K_M.gguf",
   "prompt" : "Generate a wedding greeting for my friend named XY and his wife XX.",
   "stop" : true,
   "stopped_eos" : true,
   "stopped_limit" : false,
   "stopped_word" : false,
   "stopping_word" : "",
   "timings" : {
      "predicted_ms" : 3200.376,
      "predicted_n" : 52,
      "predicted_per_second" : 16.2480908493252,
      "predicted_per_token_ms" : 61.5456923076923,
      "prompt_ms" : 675.599,
      "prompt_n" : 16,
      "prompt_per_second" : 23.6826875113788,
      "prompt_per_token_ms" : 42.2249375
   },
   "tokens_cached" : 67,
   "tokens_evaluated" : 16,
   "tokens_predicted" : 52,
   "truncated" : false
}
```

Посмотрим метрики в Grafana:
![image](https://github.com/ruslanbay/llama-kubernetes/blob/test/images/grafana-3.png)



## Удаление

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


Для сравнения Gemma потребовалось более 20 минут на то чтобы сгененрировать ответ.
