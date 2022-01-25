# Paths_To_Deploy_O-RAN
紀錄部屬O-RAN Components的各種坑

目前部屬: E-Release(2021/12釋出)
## 規格
根據OSC Bronze Release中的**Get Startted**教學中所開的規格是
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

## Non-RT RIC
## Near-RT-RIC
