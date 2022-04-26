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
![image](https://user-images.githubusercontent.com/30616512/151140124-9ed8e412-6e12-468a-8812-4b9264716768.png)
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

- Create a private docker network.

```
$ docker network create nonrtric-docker-net
```
#### **Run the Policy Management Service Docker Container**

```
$ docker run --rm -v <absolute-path-to-file>/application_configuration.json:/opt/app/policy-agent/data/application_configuration.json -p 8081:8081 -p 8433:8433 --network=nonrtric-docker-net --name=policy-agent-container nexus3.o-ran-sc.org:10002/o-ran-sc/nonrtric-a1-policy-management-service:2.3.0
```

- verify running


```
$  curl localhost:8081/a1-policy/v2/rics
```
但因為ric_simlator還沒runing起來所以預期output應該都要是 *UNAVAILABLE*

#### **Run the SDNC A1 Controller Docker Container (ONAP SDNC**)

```
$ docker-compose up
```
透過瀏覽器打開url，以確認SDNC A1 Controller 啟用並運行
http://localhost:8282/apidoc/explorer/index.html#/controller%20A1-ADAPTER-API
Username/password: admin/Kp8bJ4SXszM0WXlhak3eHlcse2gAw84vaoGGmJvUy2U

A1 Controller的 Karaf logs可以透過以下指令查看
```
$ docker exec a1controller sh -c "tail -f /opt/opendaylight/data/log/karaf.log"
```

####  **Run the Near-RT RIC A1 Simulator Docker Containers**
- RIC1
```
$ docker run --rm -p 8085:8085 -p 8185:8185 -e A1_VERSION=OSC_2.1.0 -e ALLOW_HTTP=true --network=nonrtric-docker-net --name=ric1 nexus3.o-ran-sc.org:10002/o-ran-sc/a1-simulator:2.2.0
```
- RIC2
```
$ docker run --rm -p 8086:8085 -p 8186:8185 -e A1_VERSION=STD_1.1.3 -e ALLOW_HTTP=true --network=nonrtric-docker-net --name=ric2 nexus3.o-ran-sc.org:10002/o-ran-sc/a1-simulator:2.2.0
```
- RIC3 
```
$ docker run --rm -p 8087:8085 -p 8187:8185 -e A1_VERSION=STD_2.0.0 -e ALLOW_HTTP=true --network=nonrtric-docker-net --name=ric3 nexus3.o-ran-sc.org:10002/o-ran-sc/a1-simulator:2.2.0
```
等待幾分鐘讓Policy Management Service去同步RICs
  
- verify
```
$ curl localhost:8081/a1-policy/v2/rics
```
應會回傳 *AVAILABLE*
預期輸出:
```
{"rics":[{"ric_id":"ric1","managed_element_ids":["kista_1","kista_2"],"policytype_ids":[],"state":"AVAILABLE"},{"ric_id":"ric3","managed_element_ids":["kista_5","kista_6"],"policytype_ids":[],"state":"AVAILABLE"},{"ric_id":"ric2","managed_element_ids":["kista_3","kista_4"],"policytype_ids":[""],"state":"AVAILABLE"}]}
```
- 上傳Policy Type 到ric1

```
$ curl -X PUT -v "http://localhost:8085/a1-p/policytypes/123" -H "accept: application/json" \
 -H "Content-Type: application/json" --data-binary @osc_pt1.json
```
- 上傳Policy Type 到ric3

```
$ curl -X PUT -v "http://localhost:8087/policytype?id=std_pt1" -H "accept: application/json"  -H "Content-Type: application/json" --data-binary @std_pt1.json
```
response code應該都要是201
- verify
```
$  curl localhost:8081/a1-policy/v2/policy-types
```
預期回應:
```
{"policytype_ids":["","123","std_pt1"]}
```
#### **Run the Information Coordinator Service Docker Container**

```
$ docker run --rm -p 8083:8083 -p 8434:8434 --network=nonrtric-docker-net --name=information-service-container nexus3.o-ran-sc.org:10002/o-ran-sc/nonrtric-information-coordinator-service:1.2.0
```

- verify

```
$ curl localhost:8083/data-producer/v1/info-types
```
Expected output: `[ ]`

#### **Run the Non-RT RIC Gateway**

```
$ docker run --rm -v <absolute-path-to-config-file>/application.yaml:/opt/app/nonrtric-gateway/config/application.yaml -p 9090:9090 --network=nonrtric-docker-net --name=nonrtric-gateway  nexus3.o-ran-sc.org:10002/o-ran-sc/nonrtric-gateway:1.0.0
```
- 確認是否可藉由Gateway存取到服務
```
$ curl localhost:9090/a1-policy/v2/rics
$ curl localhost:9090/data-producer/v1/info-types
```
預期output

```
{"rics":[{"ric_id":"ric1","managed_element_ids":["kista_1","kista_2"],"policytype_ids":["123"],"state":"AVAILABLE"},{"ric_id":"ric3","managed_element_ids":["kista_5","kista_6"],"policytype_ids":["std_pt1"],"state":"AVAILABLE"},{"ric_id":"ric2","managed_element_ids":["kista_3","kista_4"],"policytype_ids":[""],"state":"AVAILABLE"}]}

[ ]
```

#### **Run the Non-RT RIC Control Panel**

```
$ docker run --rm -v <absolute-path-to-config-file>/nginx.conf:/etc/nginx/nginx.conf -p 8080:8080 --network=nonrtric-docker-net --name=control-panel  nexus3.o-ran-sc.org:10002/o-ran-sc/nonrtric-controlpanel:2.3.0
```
control panel URL: http://localhost:8080/

控制面板頁面

![image](https://user-images.githubusercontent.com/30616512/151132941-19a79ebe-7396-4d0f-b7fd-2877c250dcfe.png)

可在Control Panel中對個別RIC建立不同優先級的Policy

![image](https://user-images.githubusercontent.com/30616512/151133068-a5dcc973-6272-4b33-b6cd-9b0e1c7bac8d.png)

![image](https://user-images.githubusercontent.com/30616512/151133084-13dd9743-7d06-4daa-a707-e5772f07ba44.png)


#### **Run the App Catalogue Service Docker Container**
```
$ docker run --rm -p 8680:8680 -p 8633:8633 --network=nonrtric-docker-net --name=rapp-catalogue-service nexus3.o-ran-sc.org:10002/o-ran-sc/nonrtric-r-app-catalogue:1.0.1
```
Verify that the service is up and running

```
$ curl localhost:8680/services
```

#### **Run the Helm Manager Docker Container**

```
$ cd <path-repos>/nonrtric/helm-manager
$ docker run \
    --rm  \
    -it \
    -p 8112:8083  \
    --name helmmanagerservice \
    --network nonrtric-docker-net \
    -v $(pwd)/mnt/database:/var/helm-manager-service \
    -v ~/.kube:/root/.kube \
    -v ~/.helm:/root/.helm \
    -v ~/.config/helm:/root/.config/helm \
    -v ~/.cache/helm:/root/.cache/helm \
    -v $(pwd)/config/application.yaml:/etc/app/helm-manager/application.yaml \
    nexus3.o-ran-sc.org:10002/o-ran-sc/nonrtric-helm-manager:1.1.0
```

會出現error

**Connect to localhost:3904 failed: Connection refused**

**ERROR 1 --- [pool-2-thread-1] c.a.n.c.c.i.CambriaSimplerBatchPublisher : PUB_CHRONIC_FAILURE: Send failure count is 59, above threshold 10.**

而將問題在底下留言後他們的回覆是
![image](https://user-images.githubusercontent.com/30616512/151308216-8e31ec6e-9de3-4eaa-8af2-e738d295a36c.png)
大意是:
- helm manager 在port 3904 上具有與dmaap message router的介面在
- 但這個介面沒在使用且無法disable掉
- 預設為localhost(容器內的)，因此內部的http call 在外部不可見
- 先忽略此錯誤

#### **Run the Dmaap Adaptor Service Docker Container**
建一個dmaapAdapter 目錄來放config檔
- application.yaml
  ```=
  spring:
    profiles:
      active: prod
    main:
      allow-bean-definition-overriding: true
    aop:
      auto: false
  management:
    endpoints:
      web:
        exposure:
          # Enabling of springboot actuator features. See springboot documentation.
          include: "loggers,logfile,health,info,metrics,threaddump,heapdump"
  springdoc:
    show-actuator: true
  logging:
    # Configuration of logging
    level:
      ROOT: ERROR
      org.springframework: ERROR
      org.springframework.data: ERROR
      org.springframework.web.reactive.function.client.ExchangeFunctions: ERROR
      org.oran.dmaapadapter: INFO
    file:
      name: /var/log/dmaap-adaptor-service/application.log
  server:
     # Configuration of the HTTP/REST server. The parameters are defined and handeled by the springboot framework.
     # See springboot documentation.
     port : 8435
     http-port: 8084
     ssl:
        key-store-type: JKS
        key-store-password: policy_agent
        key-store: /opt/app/dmaap-adaptor-service/etc/cert/keystore.jks
        key-password: policy_agent
        key-alias: policy_agent
  app:
    webclient:
      # Configuration of the trust store used for the HTTP client (outgoing requests)
      # The file location and the password for the truststore is only relevant if trust-store-used == true
      # Note that the same keystore as for the server is used.
      trust-store-used: false
      trust-store-password: policy_agent
      trust-store: /opt/app/dmaap-adaptor-service/etc/cert/truststore.jks
      # Configuration of usage of HTTP Proxy for the southbound accesses.
      # The HTTP proxy (if configured) will only be used for accessing NearRT RIC:s
      http.proxy-host:
      http.proxy-port: 0
    ics-base-url: https://information-service-container:8434
    # Location of the component configuration file. The file will only be used if the Consul database is not used;
    # configuration from the Consul will override the file.
    configuration-filepath: /opt/app/dmaap-adaptor-service/data/application_configuration.json
    dmaap-base-url: https://message-router:3905
    # The url used to address this component. This is used as a callback url sent to other components.
    dmaap-adapter-base-url: https://dmaapadapterservice:8435
    # KAFKA boostrap server. This is only needed if there are Information Types that uses a kafkaInputTopic
    kafka:
      bootstrap-servers: message-router-kafka:9092
  ```
- application_configuration.json(使用with karaf type)
  - application_configuration.json(without karaf type)
    ```=
    {
      "types": [
         {
            "id": "ExampleInformationType",
            "dmaapTopicUrl": "/events/unauthenticated.dmaapadp.json/dmaapadapterproducer/msgs?timeout=15000&limit=100",
            "useHttpProxy": false
         }
      ]
    }
    ```
  - application_configuration(with karaf type)
     ```=
     {
       "types": [
          {
             "id": "ExampleInformationType",
             "dmaapTopicUrl": "/events/unauthenticated.dmaapadp.json/dmaapadapterproducer/msgs?timeout=15000&limit=100",
             "useHttpProxy": false
          },
          {
           "id": "ExampleInformationTypeKafka",
           "kafkaInputTopic": "unauthenticated.dmaapadp_kafka.text",
           "useHttpProxy": false
        }
       ]
     }
     ```

Start the Dmaap Adaptor Service in a **separate shell**

```=
 docker run --rm \
-v <absolute-path-to-config-file>/application.yaml:/opt/app/dmaap-adaptor-service/config/application.yaml \
-v <absolute-path-to-config-file>/application_configuration.json:/opt/app/dmaap-adaptor-service/data/application_configuration.json \
-p 9086:8084 -p 9087:8435 --network=nonrtric-docker-net --name=dmaapadapterservice  nexus3.o-ran-sc.org:10002/o-ran-sc/nonrtric-dmaap-adaptor:1.0.0
 ```

Setup jobs to produce data according to the types in application_configuration.json

Create a file job1.json with the job definition (replace paths <url-for-jod-data-delivery> and <url-for-jod-status-delivery> to fit your environment:
 這裡替換成以下url
 
- job1.json

 ```=
 {
   "info_type_id": "ExampleInformationType",
   "job_result_uri": "http://localhost:8083/data-consumer/v1/info-jobs/job1",
   "job_owner": "job1owner",
   "status_notification_uri": "http://localhost:8434/A1-EI/v1/eijobs/job1/status",
   "job_definition": {}
 }
 ```

- Create job1 for type 'ExampleInformationType'

 ```=
 curl -X PUT -H Content-Type:application/json http://localhost:8083/data-consumer/v1/info-jobs/job1 --data-binary @job1.json
 ```

- Check that the job has been enabled - job accepted by the Dmaap Adaptor Service
原本的:

 ```=
  curl -k https://informationservice:8434/A1-EI/v1/eijobs/job1/status
  ```
但這樣會這樣:(一定還是要tls)

 ```=
  curl -k https://localhost:8434/A1-EI/v1/eijobs/job1/status
 ```

![](https://i.imgur.com/BrtdD5M.png)

karaf type

官方的url給錯，應該要是job2，不是job1

- job2.json

 ```=
 {
  "info_type_id": "ExampleInformationTypeKafka",
  "job_result_uri":"http://localhost:8083/data-consumer/v1/info-jobs/job2",
  "job_owner": "job1owner",
  "status_notification_uri": "http://localhost:8434/A1-EI/v1/eijobs/job2/status",
  "job_definition": {}
 }
 ```
- Create job2 for type 'ExampleInformationType'
 ```=
 curl -X PUT -H Content-Type:application/json http://localhost:8083/data-consumer/v1/info-jobs/job2 --data-binary @job2.json
 ```

- Check that the job has been enabled - job accepted by the Dmaap Adaptor Service
 ```
 curl -k https://localhost:8434/A1-EI/v1/eijobs/job2/status
 {"eiJobStatus":"ENABLED"}
 ```

![](https://i.imgur.com/U7BLHD8.png)

後續的 dmaapMediator Producer Container則因為port分配問題沒有run成功

可能下一版本的release會更正
 
 > Dmaap 架構: https://wiki.onap.org/pages/viewpage.action?pageId=3247130

### Ports
|Components |Port expose to localhost (http/https)|
|-----------|-------------------------------------|
|A1 Policy Management Service|8081/8443|
|App Catalogue Service|8680/8633|
|Dmaap Adaptor Service|	9087/9187|
|Dmaap Mediator Producer|9085/9185|
|Gateway|9090 (only http)|
|Helm Manager|8112 (only http)|
|Information Coordinator Service|8083/8434|
|Near-RT RIC A1 Simulator|	8085/8185,  8086/8186, 8087/8187|
|Non-RT RIC Control Panel|8080/8880|
|SDNC A1-Controller|8282/8443|


### 實際部屬情況
|Components|deployment|Description|
|---------|---------|-------------|
|App Catalogue Service*|✅||
|Dmaap Adaptor Service|✅||
|Dmaap Mediator Producer|❌|Ports are already allocated.|
|Gateway*|✅|運作正常|
|Near-RT RIC A1 Simulator|✅|運作正常|
|Non-RT RIC Control Panel|✅|運作正常|
|Policy Management Service|✅|運作正常|
|SDNC A1-Controller|✅|運作正常|

![map (6)](https://user-images.githubusercontent.com/30616512/151141547-64c91848-b332-449d-b492-ec45ffeed6f1.png)
 
 
### Policy Delivery
![Untitled Diagram drawio](https://user-images.githubusercontent.com/30616512/153117080-e3dadaea-b55d-47ed-94b5-d922188dc3be.png)


## O1 Interface
> Install : https://wiki.o-ran-sc.org/pages/viewpage.action?pageId=35881890

> Test: https://wiki.o-ran-sc.org/display/SMO/How+to+test+the+O1+interface
 
 
[o1-netconf.zip](https://github.com/leozzmc/Paths_To_Deploy_O-RAN/files/7948743/o1-netconf.zip)

```=
$ unzip o1-netconf.zip
$ cd client
$ docker-compose up -d
```
要等一陣子
 
跑完後的樣子

![image](https://user-images.githubusercontent.com/30616512/151318804-9b915436-587d-48c2-9930-5d6bfeab1a53.png)

check status

```=
docker-compose ps
```

預期輸出結果

![image](https://user-images.githubusercontent.com/30616512/151319044-2432e69b-96f7-48e1-b595-ef86dfb8ed67.png)

結果我的

![](https://i.imgur.com/Wv2cwWA.png)

沒有 sdnr、sdnrdb

突然想到之前在裝onap的時候卡住的地方

![](https://i.imgur.com/gu8CRdZ.png)

```=
kubectl describe pod dev-sdnc-0 -n onap
```

![](https://i.imgur.com/8CRbhHP.png)

port 似乎被他們佔用了
 
總之先試試url

> http://<host_ip>:8181/odlux/index.html

> username/password: admin/ Kp8bJ4SXszM0WXlhak3eHlcse2gAw84vaoGGmJvUy2U

進去了
 
![image](https://user-images.githubusercontent.com/30616512/151319631-dc7dc530-e995-4e8f-b98a-2712566b7702.png)

http://<host_ip>:8181/apidoc/explorer/index.html 這是api文件

![image](https://user-images.githubusercontent.com/30616512/151319595-a0c096c3-8b04-4937-ae6f-d5d2a5d134ea.png)
 

Add Maintainenece

![image](https://user-images.githubusercontent.com/30616512/151319568-4732a483-a62f-40f0-aa51-b91aa938d018.png)

### **O-DU Slice Assurance usecase**
> https://wiki.o-ran-sc.org/display/RICNR/O-DU+Slice+Assurance+usecase

### **O-RU Fronthaul Recovery usecase**
> https://wiki.o-ran-sc.org/display/RICNR/O-RU+Fronthaul+Recovery+usecase

 
## O1 Testing
>　https://wiki.o-ran-sc.org/display/SMO/How+to+test+the+O1+interface
 
```
git clone  https://gerrit.o-ran-sc.org/r/smo/o1.git
cd o1/
./run_tests.sh
```
 
測試完無誤後可以將 `./run_test.sh` 底下關閉服務那邊註解調，就能啟用服務了
 
 

 
## OAM

> E-Release中的OAM
> https://wiki.o-ran-sc.org/display/OAM/E-Release#ERelease-Specifications

### VES Collector 
### Installation
>　https://wiki.o-ran-sc.org/pages/viewpage.action?pageId=35881888
 
![image](https://user-images.githubusercontent.com/30616512/152935932-20925f0e-aee3-47b6-ac75-66496f61a573.png)

! 上面的 `./ves-stop.sh` 也可先註解掉
 
`./ves-start.sh` 之中的local_ip那，可自己手動key指令改

![image](https://user-images.githubusercontent.com/30616512/152935546-2ef99a14-f744-4703-aa07-952e8021f00f.png)

![image](https://user-images.githubusercontent.com/30616512/152935692-c3a1664f-aed7-4f49-b2ad-db2e92a25c34.png)

選擇第一個VM介面

![image](https://user-images.githubusercontent.com/30616512/152935796-427272e3-0f93-4668-b921-63a2306bdccc.png)

 
```
./ves_start.sh
```

 
### Testing
> https://wiki.o-ran-sc.org/pages/viewpage.action?pageId=35881892

但目前測試失敗

 
## Near-RT-RIC
非常有幫助的pdf
> https://amslaurea.unibo.it/24128/1/Malpezzi_thesis_CORRECT.pdf
 
首先需要新增一個Near-RT RIC的VM
規格: 

|資源|規格|
|----|----|
|CPU|8|
|Memory|16GB|
|Storage|160GB|
|OS|Ubuntu 18.04 LTS|

安裝必要元件 + 更新


```=
sudo -i
sudo apt-get update
sudo apt install -y vim git curl net-tools
```
 
下載repo


```=
git clone https://gerrit.o-ran-sc.org/r/dep
cd dep/
git submodule update --init --remote --recursive
```
 
安裝k8s節點

```=

cd ~/dep/tools/k8s/etc
vim infra.rc

將Kubernetes 版本改成 1.15.9


cd .. /bin
./gen-cloud-init.sh
./k8s-.......
```
 
接下來就會一直安裝k8s，完成後會重新開機

透過k8s指令來驗證是否安裝成功

```=

kubectl get pods --all-namespaces

```

接著要部屬near-rt ric 平台

```

 cd ~/dep/bin
 
./deploy-ric-platform -f ../RECIPE/PLATFORM/example_recipe_oran_e_recipe.yaml

```

接著等待部屬

接著驗證部屬是否成功

```=

kubectl get pod -n ricplt

```

![image](https://user-images.githubusercontent.com/30616512/155661066-ac40d027-ece8-4c55-bbe0-2a210c8d367c.png)

ricplatform部屬成功
 

Before any xApp can be deployed, its Helm chart must be loaded into this private Helm repository.
 

```
#Create a local helm repository with a port other than 8080 on host
docker run --rm -u 0 -it -d -p 8080:8080 -e DEBUG=1 -e STORAGE=local -e STORAGE_LOCAL_ROOTDIR=/charts -v $(pwd)/charts:/charts chartmuseum/chartmuseum:latest 

```
 
Set up the environment variables for CLI connection using the same port as used above.
 
```
export CHART_REPO_URL=http://0.0.0.0:8080
```
 
 
 
Install dms_cli tool

```
#Git clone appmgr
git clone "https://gerrit.o-ran-sc.org/r/ric-plt/appmgr"

#Change dir to xapp_onboarder
cd appmgr/xapp_orchestrater/dev/xapp_onboarder

#If pip3 is not installed, install using the following command
yum install python3-pip

#In case dms_cli binary is already installed, it can be uninstalled using following command
pip3 uninstall xapp_onboarder

#Install xapp_onboarder using following command
pip3 install ./
 
```
 
 

```
sudo chmod 755 /usr/local/bin/dms_cli
sudo chmod -R 755 /usr/local/lib/python3.6
sudo chmod -R 755 /usr/local/lib/python3.6
```
 
 clone 所要部屬xApp的 repo
 ```
 git clone "https://gerrit.o-ran-sc.org/r/ric-app/hw"
 ```
 

 ```
 dms_cli onboard CONFIG_FILE_PATH SCHEMA_FILE_PATH
 
 dms_cli onboard /root/hw/init/config-file.json /root/hw/init/schema.json
 
 ```
 
 
![image](https://user-images.githubusercontent.com/30616512/155987660-0e25ce9c-a2e9-40c7-bd1d-a69bb12ef293.png)

 
 

 ```
 curl -X GET http://localhost:8080/api/charts | jq .
 ```
 
 ![image](https://user-images.githubusercontent.com/30616512/155987523-2c69e40b-de46-4bb4-bfd7-6c15a82d8dbd.png)



 ```
 curl -X GET http://localhost:8080/api/charts/hwxapp/1.0.0
 ```
 
 ![image](https://user-images.githubusercontent.com/30616512/155988361-94e48c42-7936-452d-8601-8cadee099141.png)


 
 下載Helm Charts

 ```
 dms_cli download_helm_chart <chart_name> <version> --output_path=<PATH>
 ```
 
 ![image](https://user-images.githubusercontent.com/30616512/158763569-41341339-ac77-4294-8db5-0b2ce10cd4ab.png)

 > 參考: https://wiki.o-ran-sc.org/display/RICA/Onboard+xApp
 
 `接著需要部屬到appmgr，但還在找方法`
 
 
 接著嘗試不同xApp - Traffic Steering xApp
 
 ```
 git clone "https://gerrit.o-ran-sc.org/r/ric-app/ts"
 ```
 
 ```
 dms_cli onboard /root/ts/xapp_descriptor/config.json /root/ts/xapp-descriptor/schema.json 
 ```
 ![image](https://user-images.githubusercontent.com/30616512/158771246-98280f44-3cfc-4b4e-982e-8277acbb282b.png)

 
 列出所onboard在 local helm chart的chart

 ```
 curl -X GET http://127.0.0.1:8080/api/charts | jq .
 ```
 ![image](https://user-images.githubusercontent.com/30616512/158771584-4aa7c548-c678-478a-a218-ae00e767959d.png)

 
 ![image](https://user-images.githubusercontent.com/30616512/158790171-f4f6c7ce-af09-49da-8ced-96f1b0f3b2ee.png)

 
 提供override file

 ```
 dms_cli install_values_yaml hwxapp 1.0.0 ricxapp --output_path=.
 
 ```
 
部屬xApp
 
```
dms_cli install hwxapp 1.0.0 ricxapp --overridefile=./values.yaml
```
 
![image](https://user-images.githubusercontent.com/30616512/158814797-8bd84ed6-f3a6-4ba2-bce7-fe0a1381feeb.png)

 `好像不用 overridefile...`

檢查xapp是否被正確建立
 
 ```
 kubectl get pod -n ricxapp
 ```

![image](https://user-images.githubusercontent.com/30616512/158814998-47a55cf6-2e9c-4ce6-b8d1-862fda0e572f.png)

 `在docker login，並刪除pod重新建立replica時則運行成功`
 
 ![image](https://user-images.githubusercontent.com/30616512/158823398-cfda1cdf-973c-421b-b96c-15e6c2dcad0d.png)
 
## xApp
  
### AD
 
schema 下載網址：https://wiki.o-ran-sc.org/display/RICA/Admission+Control+xApp?preview=/3605214/7274608/adm-ctrl-xapp-schema.json
 

![12](https://user-images.githubusercontent.com/30616512/158954234-402d066b-658b-462d-bb32-11b491f17743.PNG)
 

### qp
 
```
schema path： /appmgr/xapp_orchestrater/dev/docs/xapp_onboarder/guide/embedded-schema.json

```
 
部屬成功畫面

![123456](https://user-images.githubusercontent.com/30616512/158959503-a52a1769-858f-42ad-88af-098f84293293.PNG)

 
 
 
→ 若去get pod 發現 *Container Creating* 等太久，可以做兩件事

1. docker login //因為可能是docker pull 超出限制

2. 刪掉pod，反正deployment之中的replicas會重新建立一個新的pod
 
3. 可以手動編譯 `docker build -t nexus3.o-ran-sc.org:10002/o-ran-sc/ric-app-qp:0.0.4 .`
 
可能會出現 COPY qp/ /qp Directory not found，這時就自己建一個qp/ 就好



 
## E2SIM

 ```
 git clone "https://gerrit.o-ran-sc.org/r/sim/e2-interface"
 cd e2-interface/e2sim
 ```
 
 下載dependencies
 
 ```
 sudo apt-get update
 sudo apt install -y \
   build-essential \
   git   \
   cmake \
   libsctp-dev \
   lksctp-tools \
   autoconf \
   automake \
   libtool \
   bison \
   flex \
   libboost-all-dev
 
 sudo apt-get clean
 ```
 
 buiid E2SIM
 
 ```
 mkdir build
 cd build
 cmake ..
 make package
 cmake ..  -DDEV_PKG=1
 make package
 ```
 新增環境變數
 
 ```
 vim ~/.bashrc
 
 #export E2SIM=<path_to_e2sim>
 export E2SIM=/root/e2-interface/e2sim
 ```
 
 ```
 cd  ../previous
 ./build_e2sim
 ```
 
 啟用 E2SIM
 
 ```
 cd /root/e2-interface/e2sim/previous/build 
 ./e2sim [server ip] [port]
 ex.
 ./e2sim 127.0.0.1 36422
 ```
 ![image](https://user-images.githubusercontent.com/30616512/154240212-9c93ff64-6bee-44c2-81a5-8dd2f9c25c94.png)

 教學: https://wiki.o-ran-sc.org/display/IAT/Traffic+Steering+Flows
 
 其中在做做helm install時，Helm3因改為
 
 ```
 helm install e2sim helm --name ricplt
 ```
 
 接著可在 `ricplt` namespace之中看見 e2sim pod
 
 ![image](https://user-images.githubusercontent.com/30616512/165115502-c6b7c3b6-6917-4ba5-8478-a0319b122e00.png)

 查看該pod的log
 
 ![image](https://user-images.githubusercontent.com/30616512/165115793-c9448ddf-18c5-4fca-9ca5-ea06376c19d4.png)

 ![image](https://user-images.githubusercontent.com/30616512/165115849-ba851dc0-04db-4158-8584-112f04b3cf08.png)

 ![image](https://user-images.githubusercontent.com/30616512/165115933-6b3ef25a-3678-4930-95c6-b51c3c3680ab.png)

分別為 E2SIM 到 E2TERM 的請求以及E2SIM到e2sim的回覆
 
 
 
## Network Slice Use Cases
> https://wiki.o-ran-sc.org/display/SIM/Network+Slicing+Use+Case

 
## 部屬Kpimon xApp
 

```
git clone "https://gerrit.o-ran-sc.org/r/scp/ric-app/kpimon" 
cd kpimon/
docker build .
```
 
在docker build 後會噴錯

![image](https://user-images.githubusercontent.com/30616512/165124415-e953b95a-565e-4bee-bae1-faaab29428dd.png)


Solution

![image](https://user-images.githubusercontent.com/30616512/165124525-bcd799ec-ff78-4ebe-8549-f44a81fc41e1.png)

 但解法其實更簡單，去 Dockerfile中把BaseImage的por從10004改成10002

 build途中會產生問題
 
 ![image](https://user-images.githubusercontent.com/30616512/165129287-184ef308-5849-4da1-adb4-0d8e291b1a83.png)

 Solution
 
 ![image](https://user-images.githubusercontent.com/30616512/165129343-ad008971-a356-493d-9fa5-a46b655a9618.png)

 但若直接 `docker pull  nexus3.o-ran-sc.org:10002/o-ran-sc/bldr-ubuntu20-c-go:1.0.0` 會有問題
 
 因此要根據Solution中提供的網址(為Dockerfile) 手動build並tag成 `nexus3.o-ran-sc.org:10002/o-ran-sc/bldr-ubuntu20-c-go:1.0.0`，再變更Dockerfile中的base image
 
 已經將所需的Dockerfile 放在本repo中的 `/needed_dockerfile/baseimage` 之中了
 
 這時會噴新錯誤

 ![image](https://user-images.githubusercontent.com/30616512/165138745-b80189bc-ab4c-40d6-89d5-05340c5e1509.png)

 Solution
 
 ![image](https://user-images.githubusercontent.com/30616512/165144755-93cfad95-dc0b-4487-97fb-97ef4581dfb6.png)

 
 千辛萬苦 build 完後進行 tag
 
 ![image](https://user-images.githubusercontent.com/30616512/165145333-ae7345d5-063f-46df-b194-54ddd646049e.png)

 
 
 
```
tar zxvf xappkpimon-0.2.0.tgz
cd xappkpimon/
```
 
If Helm v3

```
helm install xappkpimon . --namespace ricxapp
```
 
若刪掉xapp deployment重裝則指令改為
 
```
helm upgrade --install xappkpimon . --namespace ricxapp
 
```
 
成功
 
![image](https://user-images.githubusercontent.com/30616512/165146256-e15b81f3-a42e-49ee-8aa2-753bb5ff4ecc.png)

 
### [KPIMON部屬更新]

kpimon 部屬大全

![image](https://user-images.githubusercontent.com/30616512/165261157-59716f00-81aa-4a37-a662-1a11869bd6a9.png)
 
部屬完後接著依序部屬 `trafficxapp` , `qp-driver xapp`, `qp xapp`

需求版本
 
|xApp|version|
|----|-------|
|trafficxapp|1.2.1|
|qp-driver|1.0.9|
|qp|0.0.3|
 

版本需至每個xApp repo中的 `xapp-descriptor/config.json` 之中修改成所需版本
 
接著就依序onboard

> ⚠ 若有刪除某個 xApp的deployment，再次重新部署會噴錯(ex. ...name cannot re-use....)
這時只要 `dms_cli uninstall [xApp_name] [namespace]` 接著再次重裝就不會噴錯
 
```
dms_cli onboard [path/to/config.json] [/path/to/schema.json]
```
接著依序安裝


```
dms_cli install trafficxapp 1.2.1 ricxapp
dms_cli install qpdriver 1.0.9 ricxapp
dms_cli install qp 0.0.3 ricxapp
```

![image](https://user-images.githubusercontent.com/30616512/165294678-3abe1d9d-2cf5-4cad-b10a-b21f6879fd45.png)

### Create Policy Type
 
先建立 create.json
 

```
{"name" : "tsapolicy", "description" : "tsa parameters", "policy_type_id" : 20008, "create_schema" : { "$schema" : "http://json-schema.org/draft-07/schema#", "type" : "object", "properties" : { "threshold" : { "type" : "integer", "default" : 0 } }, "additionalProperties" : false } }
```

接著先找到 kong的svc ip

```
kubectl get svc -n ricplt | grep kong
```

上傳Policy Type

```
curl -X PUT --header "Content-Type: application/json"  --data-binary @create.json   http://<Base URL for Kong>/a1mediator/a1-p/policytypes/20008
```

倘若無法存取，可先由kong endpoint 試試

```
kubectl get ep -n ricplt | grep kong
```

![image](https://user-images.githubusercontent.com/30616512/165298142-7bef37ec-8624-430b-a5b4-9a465bfd13c3.png)
 
一樣上傳 Policy Type

```
curl -X PUT --header "Content-Type: application/json"  --data-binary @create.json   http://10.244.0.20:32080/a1mediator/a1-p/policytypes/20008
```

回傳一個 `""`
 
接著去GET看看是否有上傳成功

![image](https://user-images.githubusercontent.com/30616512/165298450-1e08ec09-789d-45a2-b2fa-a65a21c4851c.png)

 
 
 ### Create Policy
 
 ```
 curl -X PUT --header "Content-Type: application/json" --data "{\"threshold\" : 5}" http://<Base URL for Kong>/a1mediator/a1-p/policytypes/20008/policies/tsapolicy145
 ```

 成功則回傳一個 `""`
 
 
 完成後
 
 需確認 `ricplt` namespace之中有 15個Pod全部狀態為 `Running`
 
 ⛔若 `e2term` 出問題請將 e2term所有資源刪除，在去 ~/dep/ric-dep/e2term/
 
 ```
 helm install r4-e2term . --namespace ricplt
 ```
 若報錯，則可能為資源沒刪光
 
 重啟後，確認 `e2term` deployment 有跑起來
 
 接著 e2sim 應該會 `Crashloopback` ，此時也將 e2sim deployment 刪掉
 
 
 去 `/e2-interface/e2sim/e2sm_examples/kpm_e2sm` 中將 Dockfile 中的e2term-sctp-alpha service ip 改成新的ip
 
 透過 `kubectl get service -n ricplt -n ricplt` 去查詢
 
 接著重build image (原image: e2simul 也要刪除)
 
 ```
 docker rmi -f [image_name]
 cd ~/e2-interface/e2sim/e2sm_examples/kpm_e2sm 
 docker build .
 ```

 
 
 
