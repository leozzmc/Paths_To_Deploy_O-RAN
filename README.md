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
## Near-RT-RIC
