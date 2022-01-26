# Paths_To_Deploy_O-RAN
紀錄部屬O-RAN Components的各種坑

目前部屬: E-Release(2021/12釋出)
## 規格
根據OSC Bronze Release中的**Getting Started**教學中所開的規格是
```=
SMO:
- CPU: 4
- Mem: 16GB
- Storage: 160GB
- OS: Ubuntu 18.04 LTS

Near-RT RIC
- CPU: 4
- Mem: 32GB
- Storage: 160GB
- OS: Ubuntu 18.04 LTS
```
## SMO
部屬順序: K8S-Setting → ONAP → Non-RT-RIC → RIC-AUX → RIC-Infra

https://wiki.o-ran-sc.org/display/GS/SMO+Installation

### K8S-Setting
```
$ sudo -i
$ apt install -y vim curl git net-tools
$ git clone http://gerrit.o-ran-sc.org/r/it/dep
$ cd dep/
$ git submodule update --init --recursive --remote
```

```
$ cd dep/tools/k8s

$ vim etc/infra.rc

-----------------------------------------------
INFRA_DOCKER_VERSION=""
INFRA_K8S_VERSION="1.15.9"
INFRA_CNI_VERSION="0.7.5"
INFRA_HELM_VERSION="2.17.0"
-----------------------------------------------

$ cd ../bin
$ ./gen-cloud.init.sh  //會產生k8s-1node-cloud-init-.....sh 的可執行檔

$ vim k8s-1node-cloud-init-.....sh

-----------------------------------------------
elif [[ ${UBUNTU_RELEASE} == 18.* ]]; then
echo "Installing on Ubuntu $UBUNTU_RELEASE (Bionic Beaver)"
if [ ! -z "${DOCKERV}" ]; then
DOCKERVERSION="${DOCKERV}-0ubuntu1~**18.04.3**"  //改成 18.04.3
fi
-----------------------------------------------

```
完成後會重新開機
```
$ kubectl get pods --all-namespaces
```
應該會看見 kube-system namespace 底下所有Pod都要顯示  *Running*
這樣一來SMO的K8S節點設定就完成了

### ONAP

```
$ cd dep/tools/onap
$ ./install initlocalrepo
```
接著會開始一系列長久的安裝過程(3-4小時)
而做完後通常很可能會卡在
出現這樣的指令
```
0/7 SDNC-SDNR pods and 0/7 Message Router pods running
```
這時候如果去 get pod，會發現沒有pod在跑
若使用
```
$ helm list
```
會發現 charts SDNC是build failed的，而經過stackoverflow的大神們解釋

通常是做到這時，已經超出dockerhub的Pull限制了(匿名用戶每6小時只能pull 100次)

因此需要登入你的dockerHub(登入用戶每6小時能夠pull 200次)
```
$ docker login 

輸入你的username跟passwd
```
再次執行
```
$ ./install.sh   //這次不需要init repo
```
接下來的 畫面很可能是這樣
![image](https://user-images.githubusercontent.com/30616512/150942406-4368ade8-d55b-4620-9695-969da1464be5.png)
但實際去get pods，會發現其實只是pod還沒init完成或容器正在建立，大概再等一陣子就能完成(等了2小時)
```
$ kubectl get pod -n onap
```
![image](https://user-images.githubusercontent.com/30616512/150942720-e7a6fd11-15c9-490e-ae38-37343eaaad32.png)
![image](https://user-images.githubusercontent.com/30616512/150945278-2ad4e32a-e1f2-4b7b-bb63-284b443d28d5.png)

*安裝成功*

### Non-RT RIC
nonrtric的安裝方式有好幾條路
1. 透過Kubernetes裝
2. 透過docker裝

#### 透過Kubernetes
```
$ sudo /dep/bin/deploy-nonrtric -f  /dep/RECIPE/NONRTRIC/example-recipe.yaml
```
等待安裝部屬
但D Release以及E Release都會卡在這
![image](https://user-images.githubusercontent.com/30616512/151089763-7842abc3-03e8-4838-bb89-dfdb8e7d40ba.png)

#### 透過Docker-Build
> https://wiki.o-ran-sc.org/display/RICNR/Release+E+-+Build

- prerequirement

```
- docker (latest)
- docker-compose
- java 11
- Maven 3.6
- golang(v1.13.8)(v1.17)都要裝
```

- docker-compose
```
$ sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

$ sudo chmod +x /usr/local/bin/docker-compose
$ docker-compose
```
若docker-compose沒反應
```
$ sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```
若還是沒反應，就手動去網頁仔下來，然後放到 /usr/local/bin，檔名取為docker-compose

- java 11
> https://www.cjavapy.com/article/90/

```
# Build from binary
$ wget https://download.java.net/java/GA/jdk11/13/GPL/openjdk-11.0.1_linux-x64_bin.tar.gz
# Build from apt
$ sudo apt install openjdk-11-jre-headless

$ mkidir /usr/local/jdk
$ tar -zxf openjdk-11.0.1_linux-x64_bin.tar.gz -C /usr/local/jdk

#設定Java_HOME
$ echo "JAVA_HOME=/usr/local/jdk/jdk-11.0.1" | sudo tee -a ~/.bash_profile\
&& echo "export JAVA_HOME" \
| sudo tee -a ~/.bash_profile

$ echo $JAVA_HOME

$ echo "PATH=$PATH:$JAVA_HOME/bin" | sudo tee -a ~/.bash_profile \
&& "export PATH" | sudo tee -a ~/.bash_profile && source ~/.bash_profile

$ java --version

```

- maven

```
$ sudo apt install maven
```
- [settings.xml](https://git.onap.org/oparent/plain/settings.xml)

將檔案存成settings.xml，存到 `~/.m2/settings.xml`

- golang
[教學](https://blog.csdn.net/qq_33867131/article/details/106853024)
```=
--ver: 1.13.8---
$ wget https://go.dev/dl/go1.13.8.linux-amd64.tar.gz
$ tar -zxvf go1.13.8.linux-amd64.tar.gz -C /usr/local


$ sudo vim /etc/profile
# 在最後面加上路徑
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export GOBIN=$GOPATH/bin
export PATH=$GOPATH:$GOBIN:$GOROOT/bin:$PATH

$ source /etc/profile
$ sudo vim ~/.bashrc
# 最下面加上
source /etc/profile

#重新載入bashrc
$ .~/.bashrc

$ go version

---ver: 1.17---
$ wget https://go.dev/dl/go1.17.linux-amd64.tar.gz
$ cd /usr/local
$ mkdir go-1.17
$ tar -zxvf go1.17.linux-amd64.tar.gz -C /usr/local/go-1.17

#修改/etc/profile
加上 export GOROOT=/usr/local/go-1.17/go
$ source /etc/profile
$ go version
```
在編譯nonrtric container時，會出現go版本問題
解法: 編譯卡在哪個版本就改環境變數再去source，來切換版本
最終就能成功

-  Build Code
```=
git clone "https://gerrit.o-ran-sc.org/r/nonrtric"

cd nonrtric
# 這裡開始會碰到超多錯誤
mvn clean install -Dmaven.test.skip=true
```
最多出錯的地方會是 /nonrtric/dmaap-mediator-producer/

要去/stub內去每個裡面都下`go build` 以及 `go mod tidy`

![](https://i.imgur.com/XDf3vz6.png)

這時候可能會需要去 /etc/profile去切換go的版本，記得切換要用source指令

最後出來到/dmaap-mediator中 `go-build` 以及 `go mod tidy`

其實在這可以手動build /nonrtric/dmaap-mediator-producer/build_and_test.sh

成功畫面:

![](https://i.imgur.com/ktvJjII.png)

最後檢查 docker images
![image](https://user-images.githubusercontent.com/30616512/151091603-4299cbce-e16b-485b-9f40-662f8ad9da7e.png)

#### NonRTRIC Docker Image List(E Release)
|Components|Release Image and Version Tag|Manual snapshot and version tag|
|---------|---------|--------|
|App Catalogue Service*|nexus3.o-ran-sc.org:10002/o-ran-sc/nonrtric-r-app-catalogue:1.0.1|	o-ran-sc/nonrtric-r-app-catalogue:1.1.0-SNAPSHOT|
|Dmaap Adaptor Service|nexus3.o-ran-sc.org:10002/o-ran-sc/nonrtric-dmaap-adaptor:1.0.0|o-ran-sc/nonrtric-dmaap-adaptor:1.0.0-SNAPSHOT|
|Dmaap Mediator Producer|nexus3.o-ran-sc.org:10002/o-ran-sc/nonrtric-dmaap-mediator-producer:1.0.0|o-ran-sc/nonrtric-dmaap-mediator-producer:1.0.0-SNAPSHOT|
|Gateway*|nexus3.o-ran-sc.org:10002/o-ran-sc/nonrtric-gateway:1.0.0|o-ran-sc/nonrtric-gateway:1.1.0-SNAPSHOT|
|Helm Manager|nexus3.o-ran-sc.org:10002/o-ran-sc/nonrtric-helm-manager:1.1.0|o-ran-sc/nonrtric-helm-manager:1.1.0-SNAPSHOT|
|Information Coordinator Service|nexus3.o-ran-sc.org:10002/o-ran-sc/nonrtric-information-coordinator-service:1.2.0|o-ran-sc/nonrtric-information-coordinator-service:1.2.0-SNAPSHOT|
|Near-RT RIC A1 Simulator|nexus3.o-ran-sc.org:10002/o-ran-sc/a1-simulator:2.2.0|o-ran-sc/a1-simulator:latest|
|Non-RT RIC Control Panel|nexus3.o-ran-sc.org:10002/o-ran-sc/nonrtric-controlpanel:2.3.0|o-ran-sc/nonrtric-controlpanel:2.3.0-SNAPSHOT|
|Policy Management Service|nexus3.o-ran-sc.org:10002/o-ran-sc/nonrtric-a1-policy-management-service:2.3.0|o-ran-sc/nonrtric-a1-policy-management-service:2.3.0-SNAPSHOT|
|SDNC A1-Controller|nexus3.onap.org:10002/onap/sdnc-image:2.2.3|Use release version|

- Build near-rt-ric-simulator container

```
$ git clone "https://gerrit.o-ran-sc.org/r/sim/a1-interface"
$ cd a1-interface/near-rt-ric-simulator
$ docker build -t near-rt-ric-simulator:latest .
```

- Build nonrtric / Control panel and gateway containers

```
$ git clone "https://gerrit.o-ran-sc.org/r/portal/nonrtric-controlpanel" -b e-release
$ cd nonrtric-controlpanel
$ cd nonrtric-gateway
$ mvn clean install  -Dmaven.test.skip=true
$ docker build --build-arg JAR=nonrtric-gateway-1.1.0-SNAPSHOT.jar -t o-ran-sc/nonrtric-gateway:1.1.0-SNAPSHOT .
 
$ cd ../webapp-frontend
$ docker build -t o-ran-sc/nonrtric-controlpanel:2.3.0-SNAPSHOT .
```
最後透過 `docker images ` 檢查docker可用image


#### 透過Docker-Run
> https://wiki.o-ran-sc.org/display/RICNR/Release+E+-+Run+in+Docker



## Near-RT-RIC
這份pdf救了我
> https://amslaurea.unibo.it/24128/1/Malpezzi_thesis_CORRECT.pdf
