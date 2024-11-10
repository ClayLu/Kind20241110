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


