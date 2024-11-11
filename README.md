# Kind20241110

![image](https://github.com/ClayLu/Kind20241110/blob/main/Architecture.png)

## 用Kind建立K8S Cluster
用以下語法建立K8S Cluster
```
kind create cluster --config=kind_cluster.yml
```
其內容為:
1.建立1 control-plane和3 worker的cluster，並且在worker加上label以區分是infra或ap用
2.在infra worker設定nodeport 30090讓Kind內部的prometheus可透過infra container讓外部連線

## 安裝Calico
因為接下來要安裝MetalLB，所以不能用Kind預設的網路元件，要自行安裝相容的，這邊選擇安裝Calico

1.下載Calico
```
curl https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/calico.yaml -O
```

2.修改內容
```
      containers:
        # Runs calico-node container on each Kubernetes node. This
        # container programs network policy and routes on each
        # host.
        - name: calico-node
          image: docker.io/calico/node:v3.20.1
          env:
            # Disable the Calico RPF check
            - name: FELIX_IGNORELOOSERPF
              value: "true"
```
3.
```
kubectl apply -f calico.yaml
```

## 安裝MetalLB

1.用Helm安裝MetalLB，下載helm values
```
helm repo add metallb https://metallb.github.io/metallb
helm show values metallb/metallb > metallb.yaml
```

2.修改metallb.yaml如下，才會部屬在infra node
```
  nodeSelector:
    tier: infra
```

3.安裝MetalLB
```
helm upgrade --install metallb metallb/metallb -f metallb.yaml
```

## install prometheus

1.下載helm values
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm show values prometheus-community/kube-prometheus-stack > kube-prometheus-stack-values.yaml
```

2.安裝 Prometheus, kube-state-metrics 在infra node 節點上
```
  prometheusSpec
...
    nodeSelector:
      tier: infra
...

kube-state-metrics:
  nodeSelector:
    tier: infra
```

3.關閉grafana
```
grafana:
  enabled: false
```

4.為了要讓prometheus能連線etcd，需做以下調整
(1)etcd對外連線Port為2379，故調整prometheus的設定
```
kubeEtcd
...
  service:
    enabled: true
    port: 2379
    targetPort: 2379
```
(2)因為etcd需要key才能連線，故設定如下
先連入control-plane docker container，把etcd相關key建立secret
```
kubectl  create secret generic etcd-client-cert --from-file=/etc/kubernetes/pki/etcd/ca.crt --from-file=/etc/kubernetes/pki/etcd/healthcheck-client.crt --from-file=/etc/kubernetes/pki/etcd/healthcheck-client.key
```

修改kube-prometheus-stack-values.yaml，去讀取掛載後的etcd key
```
kubeEtcd
...
    proxyUrl: ""
    #scheme: http
    #insecureSkipVerify: false
    #serverName: ""
    #caFile: ""
    #certFile: ""
    #keyFile: ""
    scheme: https
    insecureSkipVerify: false
    serverName: localhost
    caFile: /etc/prometheus/secrets/etcd-client-cert/ca.crt
    certFile: /etc/prometheus/secrets/etcd-client-cert/healthcheck-client.crt
    keyFile: /etc/prometheus/secrets/etcd-client-cert/healthcheck-client.key
```
讓prometheus掛載該secret
```
  prometheusSpec:
...
    ##
    secrets: ['etcd-client-cert']
```
讓prometheus的service account有權限掛載secret
```
prometheus:
...
  additionalRulesForClusterRole: #[]
    - apiGroups: [ "" ]
      resources:
        - secrets
      verbs: [ "*" ]
```

(3)改為NodePort才能讓Kind外連線
```
prometheus:
...
    type: NodePort
```

(4)部屬
```
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack -f kube-prometheus-stack-values.yaml
```

## 在Kind外部屬grafana
(1)
執行Grafana docker並export port 3000和掛載硬碟讓資料持久
```
docker run -d -p 3000:3000 --name=grafana \                        
  --volume grafana-storage:/var/lib/grafana \
  grafana/grafana
```
(2)
可以從本機的http://localhost:3000/login登入
(3)
找到infra node ip搭配prometheus的nodeport，設定datasource為prometheus
```
http://192.168.228.4:30090/
```
(4)
將src/Dashboard中所有檔案import到Grafana dashboard中

## 安裝app和hpa

(1)安裝metric server
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

kubectl patch -n kube-system deployment metrics-server --type=json \
  -p '[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```
(2)把image load到kind中各node中
```
kind load docker-image  docker-demo:latest --name mykind1
```
(3)部屬app deployment和hpa，記得要調整imagePullPolicy
```
imagePullPolicy: IfNotPresent
```

## 11/11補充 搜集cAdvisor資料(包含throttled資料)

(1)prometheus這段註解
```
    cAdvisorMetricRelabelings:
      # Drop less useful container CPU metrics.
      #- sourceLabels: [__name__]
      #  action: drop
      #  regex: 'container_cpu_(cfs_throttled_seconds_total|load_average_10s|system_seconds_total|user_seconds_total)'
      # Drop less useful container / always zero filesystem metrics.
```

(2)Pod要設定resource的request和limit

(3)確認Prometheus的endpoint https://x.x.x.x:10250/metrics/cadvisor 是up，而且相關metric有進來
