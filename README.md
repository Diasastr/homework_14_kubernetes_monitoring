Налаштування Kubernetes Кластера на AWS
=======================================
### З запізненням через технічну помилку, я надто пізно помітила що не закомітила readme

Крок 1: Створення Екземплярів EC2
---------------------------------

Створила 4 екземпляри EC2 на AWS з наступними характеристиками:

-   AMI: Amazon Linux (старіша версія)
-   Тип: t3.medium
-   Кількість екземплярів: 4 (1 master, 2 workers, 1 monitoring)
-   User Data:
   ```bash
    #!/bin/bash

    # Вимкнути Swap
    sudo swapoff -a
    sudo sed -i '/swap/d' /etc/fstab
    
    # Оновити всі встановлені пакети
    sudo yum update -y
    
    # Встановити Docker
    sudo yum install docker -y
    sudo service docker start
    sudo usermod -a -G docker ec2-user
    
    # Переконатися, що параметр net.bridge.bridge-nf-call-iptables встановлений на 1
    sudo tee /etc/sysctl.d/k8s.conf '<<EOF
    net.bridge.bridge-nf-call-iptables = 1
    EOF'
    sudo sysctl --system
    
    # Вимкнути SELinux
    sudo setenforce 0
    sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
    
    # Встановити kubeadm, kubelet та kubectl
    cat '<<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
    enabled=1
    gpgcheck=1
    repo_gpgcheck=1
    gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
    exclude=kube*
    EOF'
    
    sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
    sudo systemctl enable --now kubelet
    
    # Встановити Telnet
    sudo yum install telnet -y
   ```

Крок 2: Ініціалізація Kubernetes на Master Node
-----------------------------------------------

На master-ноді запустіть:

```bash

# Ініціалізувати кластер Kubernetes (виконувати це тільки на вузлі-мастері)
kubeadm init --pod-network-cidr=10.244.0.0/16
# Не забути зберегти команду приєднання для додавання вузлів-робітників.

# Виконати кроки з респонсу від ініту - зберегти конфіг файл і втановити мережевий плагін CNI, наприклад, Calico
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yam
```

Крок 3: Приєднання Worker Nodes
-------------------------------

На кожному worker-ноді запустила команду, яку надає `kubeadm init` після ініціалізації:

```bash

kubeadm join <your-master-node-ip>:<port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>

```


Крок 4: Маркування Monitoring Node
----------------------------------

Маркуємо monitoring node за допомогою:

```bash

kubectl label nodes <your-monitoring-node-name> monitoring=enabled
```


Крок 5: Додавання Додаткових Сервісів (Web і Database)
------------------------------------------------------

1.  Web Service Deployment і Database Service Deployment з дз 13



Крок 6: Налаштування Моніторингу
--------------------------------

Використала файл нижче для розгортання Prometheus, Grafana, Loki, Node Exporter, і Promtail:

```yaml
# Prometheus ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  labels:
    app: prometheus
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
      - job_name: 'kubernetes-nodes'
        kubernetes_sd_configs:
          - role: node
        relabel_configs:
          - action: labelmap
            regex: __meta_kubernetes_node_label_(.+)
---
# Prometheus Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
spec:
  selector:
    matchLabels:
      app: prometheus
  replicas: 1
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: config-volume
          mountPath: /etc/prometheus
      volumes:
      - name: config-volume
        configMap:
          name: prometheus-config
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: monitoring
                operator: In
                values:
                - enabled
---
# Grafana Deployment with Prometheus Data Source
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
spec:
  selector:
    matchLabels:
      app: grafana
  replicas: 1
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana
        ports:
        - containerPort: 3000
        volumeMounts:
        - name: grafana-datasources
          mountPath: /etc/grafana/provisioning/datasources
      volumes:
      - name: grafana-datasources
        configMap:
          name: grafana-datasources-config
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: monitoring
                operator: In
                values:
                - enabled
---
# Grafana DataSources ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources-config
data:
  prometheus.yaml: |
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        access: proxy
        url: http://prometheus:9090
---
# Loki ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: loki-config
data:
  loki-config.yaml: |
    auth_enabled: false
    server:
      http_listen_port: 3100
    ingester:
      lifecycler:
        address: 127.0.0.1
        ring:
          kvstore:
            store: inmemory
          replication_factor: 1
      chunk_idle_period: 3m
      chunk_retain_period: 1m
    schema_config:
      configs:
        - from: 2020-10-24
          store: boltdb-shipper
          object_store: filesystem
          schema: v11
          index:
            prefix: index_
            period: 24h
    storage_config:
      boltdb_shipper:
        active_index_directory: /tmp/loki/index
        cache_location: /tmp/loki/cache
        cache_ttl: 24h
        shared_store: filesystem
      filesystem:
        directory: /tmp/loki/chunks
---
# Loki Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: loki
spec:
  selector:
    matchLabels:
      app: loki
  replicas: 1
  template:
    metadata:
      labels:
        app: loki
    spec:
      containers:
      - name: loki
        image: grafana/loki:latest
        ports:
        - containerPort: 3100
        volumeMounts:
        - name: config
          mountPath: /etc/loki
      volumes:
      - name: config
        configMap:
          name: loki-config
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: monitoring
                operator: In
                values:
                - enabled
---
# Node Exporter DaemonSet
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true
      containers:
      - name: node-exporter
        image: prom/node-exporter
        ports:
        - containerPort: 9100
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: monitoring
                operator: In
                values:
                - enabled
---
# Promtail ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: promtail-config
data:
  config.yml: |
    server:
      http_listen_port: 9080
    positions:
      filename: /tmp/positions.yaml
    clients:
      - url: http://loki:3100/loki/api/v1/push
    scrape_configs:
    - job_name: system
      static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /var/log/*log
---
# Promtail DaemonSet
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: promtail
spec:
  selector:
    matchLabels:
      name: promtail
  template:
    metadata:
      labels:
        name: promtail
    spec:
      containers:
      - name: promtail
        image: grafana/promtail:latest
        args:
        - "-config.file=/etc/promtail/config.yml"
        volumeMounts:
        - name: config
          mountPath: /etc/promtail
        - name: logs
          mountPath: /var/log
      volumes:
      - name: config
        configMap:
          name: promtail-config
      - name: logs
        hostPath:
          path: /var/log
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: monitoring
                operator: In
                values:
                - enabled
```
Потім задеплоїла:
```bash
kubectl apply -f complete-monitoring-stack.yaml
```

Крок 7: Перевірка Стану Компонентів
-----------------------------------

-   Перевірка Nodes:

    ```bash

    `kubectl get nodes`
    ```

-   Перевірка Pods:

    ```bash

    `kubectl get pods --all-namespaces`
    ```

-   Перевірка Стану Prometheus:

    Відкрила Prometheus у браузері: `http://<monitoring-node-ip>:9090`

-   Перевірка Стану Grafana:

    Відкрила Grafana у браузері: `http://<monitoring-node-ip>:3000`

-   Перевірка Метрик Node Exporter:

    Відкрийте Node Exporter метрики: `http://<node-ip>:9100/metrics`

-   Перевірка Логів Loki через Grafana:

    У Grafana, налаштувала Loki як джерело логів і переглянула їх.
