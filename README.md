# Jenkins Slave in Minikube
場景:
1. 本機為 Mac OS
2. 本機以 `docker run` 啟動 Jenkins (Master)
3. 本機以 minikube 建造 kubernetes 集群
4. Jenkins Agent(Slave) 是以 Pod 的形式，產生在 minikube 內，建構完隨即刪除
5. 佈署應用程式 Nginx 也是佈署在相同 minikube 中

## Minikube
在 localhost 啟動 minikube
```
minikube start
```

### Kubernetes URL
啟動後

1. 由 `docker ps` 可以得知 minikube API server **8443** port mapping 為 **127.0.0.1:32772** (**埠號不一定**)

![](https://i.imgur.com/uGZehFG.png)

2. 由 `minikube node list`  可以得知 minikube API server 所在的 IP 位址為 **192.168.49.2**

![](https://i.imgur.com/NV9yhpv.png)



### Generate PFX
以 minikube 原有的密鑰檔及CA憑證（如下），產生 PFX 檔，將用於與 K8s API Server 的溝通（設定 Jenkins Credential）
* apiserver.key
* apiserver.crt
* ca.crt

```
openssl pkcs12 -export \
  -out ~/.minikube/profiles/minikube/minikube.pfx \
  -inkey ~/.minikube/profiles/minikube/apiserver.key \
  -in ~/.minikube/profiles/minikube/apiserver.crt \
  -certfile ~/.minikube/ca.crt -passout pass:09350935
```


### ConfigMap From Files
將後續 apply YAML 所需的 kubeconfig 相關的 crt & key（如下），以 ConfigMap 的形式儲存
* client.key
* client.crt
* ca.crt


```
kubectl create configmap minikubeconfig \
  --from-file=/Users/eric/.minikube/ca.crt \
  --from-file=/Users/eric/.minikube/profiles/minikube/client.crt \
  --from-file=/Users/eric/.minikube/profiles/minikube/client.key
```

### Kubernetes Dashboard
可以 `minikube dashboard` 啟動 Kubernetes Dashboard




## Jenkins

在 localhost 啟動 Jenkins docker container
```
docker run --name local-jenkins -p 8080:8080 -p 50000:50000 -v `pwd`/home:/var/jenkins_home -d --rm jenkins/jenkins:lts
```

啟動後，由 `docker ps` 可以得知 jenkins **8080** port mapping 為 **0.0.0.0:8080**

![](https://i.imgur.com/uGZehFG.png)



### Plugins
* Kubernetes plugin
* Kubernetes Continuous Deploy Plugin

### Credentials

#### for Kubernetes Cloud
* Kind: Certificate
* Certificate: [選擇 PFX 檔]
* Password: [輸入 PFX 密碼]
* ID: minikubeconfig

![](https://i.imgur.com/4Ng7yhc.png)

#### for apply YAML
* Kind: Kubernetes configuration (kubeconfig)
* ID: mykubeconfig
* Kubeconfig: [選擇 Enter directly，將 ~/.kube/config 內容貼上，如下] (註3)

```yaml=
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /home/jenkins/minikube/ca.crt
    server: https://192.168.49.2:8443
  name: minikube
contexts:
- context:
    cluster: minikube
    namespace: default
    user: minikube
  name: minikube
current-context: minikube
kind: Config
preferences: {}
users:
- name: minikube
  user:
    client-certificate: /home/jenkins/minikube/client.crt
    client-key: /home/jenkins/minikube/client.key
```

### Kubernetes Cloud
Config Clouds > 新增雲，選擇 Kubernetes

#### Kubernetes
* Name: kubernetes
* Kubernetes URL: https://host.docker.internal:32772 (註1)
* Kubernetes server certificate key: [將 ca.crt 內容貼上]
* Disable https certificate check: 勾選
* Credentials: [選擇 minikubeconfig Credential]
* Jenkins URL: http://host.docker.internal:8080/ (註1)
* Jenkins tunnel: host.docker.internal:50000 (註1)

#### Pod Template
* Name: kube
* Labels: kubepod
* Containers
    * Name: jnlp-slave
    * Docker image: jenkins/jnlp-slave:latest
    * Working directory: /home/jenkins/
* Volumes: 
    * Config Map name: minikubeconfig
    * Mount path: /home/jenkins/minikube/

![](https://i.imgur.com/ZycT7IC.png)
![](https://i.imgur.com/jnhpiOL.png)
![](https://i.imgur.com/kkpVYkn.png)
![](https://i.imgur.com/S1OyEYv.png)
![](https://i.imgur.com/jXJvL0W.png)
![](https://i.imgur.com/PtiiXWq.png)



### Jobs

Pipeline script from SCM
https://github.com/eric-systex/playjenkins.git (branch:test-deploy-stage)

![](https://i.imgur.com/feCRMe4.png)


Jenkinsfile (註2)

```groovy=
pipeline {

  agent { label 'kubepod' }

  stages {

    stage('Checkout Source') {
      steps {
        git url:'https://github.com/eric-systex/playjenkins.git', branch:'test-deploy-stage'
      }
    }

    stage('Deploy App') {
      steps {
        script {
          kubernetesDeploy(configs: "nginx.yaml", kubeconfigId: "mykubeconfig")
        }
      }
    }

  }

}
```


nginx.yaml

```yaml=
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: web
  name: mynginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - image: nginx
        name: mynginx

---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: web
  name: mynginx
spec:
  ports:
  - nodePort: 32223
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: web
  type: NodePort
```




## 備註

* 註1: **host.docker.internal** 在 docker container 內會被解析為 Host 的 **127.0.0.1** (來源: https://stackoverflow.com/a/24326540)
* 註2: agent label 必須和 Pod Template Labels 一致
* 註3: 若設定 minikube:8443 會有 Hostname minikube not verified 的問題



## 參考

* https://www.youtube.com/watch?v=V4kYbHlQYHg
* https://github.com/justmeandopensource/playjenkins/tree/test-deploy-stage
* https://plugins.jenkins.io/kubernetes/
* https://stackoverflow.com/a/24326540
* https://stackoverflow.com/questions/40720979/how-to-access-kubernetes-api-when-using-minkube
* https://ithelp.ithome.com.tw/articles/10193237
* https://ithelp.ithome.com.tw/articles/10193489
* https://hub.docker.com/r/jenkinsci/jnlp-slave/
