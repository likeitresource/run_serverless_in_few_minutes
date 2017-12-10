![serverless](https://github.com/likeitresource/run_serverless_in_few_minutes/blob/master/images/serverless.png)

# 10分鐘建置 Serverless 服務

有時只想後端有個服務接口，來接收外部的請求。於是就開始安裝OS、建置運作環境、維護環境...:expressionless:

更別說還是不時為這個環境上Patch...:confounded:

Serverless 是抽象概念，並不是說沒有 Server了，而是讓使用者可以更專注於功能品質，免於維護服務器。Serverless 是一個 event-driven architectures，開發人員可以開發許多功能，提供外部事件來觸發。例如 CI/CD pipeline，或是現在物聯網世界內的小 Sensor。

本文為一個快速引導，使用 [OpenFaaS](https://github.com/openfaas/faas) 來建置、實作一個基礎的產生 OTP Code Serverless範例。

## 前置作業: 準備一台主機或是虛擬機

主機可以為實體機或是虛擬機，本文使用 VM，提供 1Core, 2G RAM, 20G HD。

作業系統使用 [ubuntu 16.04](http://releases.ubuntu.com/16.04/)。安裝完使用以下指令更新 OS。

```shellscrupt
$ sudo apt-get update && sudo apt-get dist-upgrade
```

以下是我們的建置步驟:

* 安裝 Docker && 建置 Docker Swarm
* 部署 OpenFaaS
* 撰寫 Function (使用 Python)
* 執行 Function
* 觀察監控數據

## 安裝 Docker && 建置 Docker Swarm

OpenFaaS 使用 Container 提供 function 運行環境，底層可透過 [Docker Swram](https://docs.docker.com/engine/swarm/swarm-tutorial/create-swarm/) 或 [Google Kubernetes](https://kubernetes.io/) 來延伸資源規模。

本文為一個快速導引，使用 Docker Swarm 來快速建置 OpenFaaS 所需的 Container 環境。

Docker 安裝很簡單，只需:

```shellscrupt
$ wget -O - https://get.docker.com | sudo bash
```
再去泡杯茶 :tea:

Docker安裝完，可以使用以下指令，將 linux 的使用者 (user) 加入 docker group，這樣一來，不用 root 權限即可以使用 Docker。

```
$ sudo usermod -aG docker YOUR_USER_NAME
```

接下建置 docker swarm

```
$ docker swarm init
Swarm initialized: current node (kl74eq0mwhm03xtt8c6m95253) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-4m0wb27fsf8wsvfzrz60cjiuumkojzf3dbcn29vj7dufz9bevv-bezhrnutrcbmnbwk16xcafo9h 192.168.64.129:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

本文只使用一台VM，身兼 manager & worker。若環境中有多台機器，可以登入到其他機器，並使用上述的 docker swarm join --token xxxx，讓加入更多機器。

完成後，執行 "docker node ls"，看到以下畫面，就表示可以繼續下一步驟。

```
docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
kl74eq0mwhm03xtt8c6m95253 *   ubuntu              Ready               Active              Leader
```
## 部署 OpenFaaS

執行以下指令，會自動部署 OpenFaaS。 (過程需要 git，可以使用 sudo apt-get install git 安裝)

```
$ cd ~
$ git clone https://github.com/alexellis/faas/ && cd faas
$ ./deploy_stack.sh
Deploying stack
Creating network func_functions
Creating service func_decodebase64
Creating service func_nodeinfo
Creating service func_echoit
Creating service func_alertmanager
Creating service func_base64
Creating service func_prometheus
Creating service func_markdown
Creating service func_webhookstash
Creating service func_gateway
Creating service func_hubstats
Creating service func_wordcount
```

完成後，透過 docker service ls 可觀察完成情況。每一個 service 的 REPLICAS，都至少啟動1個，即 1/1。

```
docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                              PORTS
bgn7f2g1okop        func_alertmanager   replicated          1/1                 functions/alertmanager:latest
0ymyrj6gg60y        func_base64         replicated          1/1                 functions/alpine:latest
5ouo3f1bp9ly        func_decodebase64   replicated          1/1                 functions/alpine:latest
ul7oapw00ek3        func_echoit         replicated          1/1                 functions/alpine:latest
b2eyfpp3tv9z        func_gateway        replicated          1/1                 functions/gateway:0.6.13           *:8080->8080/tcp
c68p1dgibn7x        func_hubstats       replicated          1/1                 functions/hubstats:latest
ejqw3aal9co0        func_markdown       replicated          1/1                 functions/markdown-render:latest
kc3jyppvavfj        func_nodeinfo       replicated          1/1                 functions/nodeinfo:latest
3c9vovz3u6tg        func_prometheus     replicated          1/1                 functions/prometheus:latest        *:9090->9090/tcp
e4j66chueiah        func_webhookstash   replicated          1/1                 functions/webhookstash:latest
43c2tya7rbus        func_wordcount      replicated          1/1                 functions/alpine:latest
```

使用瀏覽器，打開 http://your_host_ip:8080/

就會看到 OpenFaaS Portal (以下畫面)；左邊列表，是內建的 function 範例。

![serverless](https://github.com/likeitresource/run_serverless_in_few_minutes/blob/master/images/built-in-example.png)


為了後續建置 function，還需要安裝 faas-cli。安裝方式還是很簡單:
```
$ curl -sSL https://cli.openfaas.com | sudo sh
x86_64
Getting package https://github.com/openfaas/faas-cli/releases/download/0.5.0/faas-cli
Attemping to move faas-cli to /usr/local/bin
New version of faas-cli installed to /usr/local/bin
Creating alias 'faas' for 'faas-cli'.
```

## 撰寫 Function (使用 Python)
基本環境都完成了，讓我們開始撰寫 function (本文使用 Python，若對python有興趣，可以加入 [FB公開社團 Python Taiwan](https://www.facebook.com/groups/pythontw/) 

本範例提供一個產生 otp 密碼服務。先建立一個目錄，並使用上一步驟安裝的 faas-cli 來自動產生相關樣本
```
$ cd ~ && mkdir -p funcs && cd funcs
$ faas-cli new --lang python my-otp
2017/12/10 01:42:08 No templates found in current directory.
2017/12/10 01:42:08 HTTP GET https://github.com/openfaas/faas-cli/archive/master.zip
2017/12/10 01:42:11 Writing 287Kb to master.zip

2017/12/10 01:42:11 Attempting to expand templates from master.zip
2017/12/10 01:42:11 Fetched 10 template(s) : [csharp go-armhf go node-arm64 node-armhf node python-armhf python python3 ruby] from https://github.com/openfaas/faas-cli
2017/12/10 01:42:11 Cleaning up zip file...
Folder: my-otp created.
  ___                   _____           ____
 / _ \ _ __   ___ _ __ |  ___|_ _  __ _/ ___|
| | | | '_ \ / _ \ '_ \| |_ / _` |/ _` \___ \
| |_| | |_) |  __/ | | |  _| (_| | (_| |___) |
 \___/| .__/ \___|_| |_|_|  \__,_|\__,_|____/
      |_|


Function created in folder: my-otp
Stack file written: my-otp.yml
```

資料夾，會發現產生了兩個目錄，兩個檔案。其中 template 為樣本庫，暫時先不討論它。
我們直接來看:
```
my-otp.yml  # function 相關屬性
my-otp\handler.py # 實作程式碼的地方
my-otp\requirements.txt # 若需要 python third-package，填入此處
```

請編輯 my-otp\handler.py，貼入以下 sample code
```python
import time
from hashlib import sha512

def handle(user_id):
    t_time = int(time.time())
    random_code = user_id[::-1]
    otp = sha512(str(t_time)[:-1] + random_code).hexdigest()[:6]
    print(otp)
```

建置 function
```
$ faas-cli build -f my-otp.yml
```

會看到開始 build docker image~
```
[0] > Building: my-otp.
Clearing temporary build folder: ./build/my-otp/
Preparing ./my-otp/ ./build/my-otp/function
Building: my-otp with python template. Please wait..
Sending build context to Docker daemon  7.168kB
Step 1/16 : FROM python:2.7-alpine
2.7-alpine: Pulling from library/python
ab7e51e37a18: Pull complete
4a57a4e05b89: Pull complete
de1aaf39fd2e: Downloading [=====================>                             ]  8.747MB/20.11MB
275f7596216d: Download complete
```

使用 docker images 查看，若確認有 my-otp即可以往下一步驟
```
$ docker images | grep my-otp
my-otp                           latest              6ee151252da2        2 minutes ago       81.1MB
```

## 執行 Function
讓我們來測試這個 otp服務。

測試前，需要先將部署 function。(記得加入 --gateway，來指定 Gateway，並將 "192.168.64.129" 改為實際 IP

```
$ faas-cli deploy -f my-otp.yml --gateway http://192.168.64.129:8080
Deploying: my-otp.
No existing function to remove
Deployed.
URL: http://192.168.64.129:8080/function/my-otp

200 OK
```

使用 faas-cli list，若 my-opt Replicas == 1，表示部署成功，可以提供服務了 :+1:
```
$ faas-cli list --gateway http://192.168.64.129:8080
Function                        Invocations     Replicas
...
my-otp                          0               1
...
```

先用 curl 測試， '-d "jess"' 表示要產生哪一位使用者的 otp
```
$ curl -XPOST 192.168.64.129:8080/function/my-otp -d "jess"
```

回到 OpenFaaS Portal，會看到 my-otp 已經在列表上。請依照下圖步驟，逐一輸入執行。步驟3，即是發出 request，步驟4即是回傳值。
![serverless](https://github.com/likeitresource/run_serverless_in_few_minutes/blob/master/images/otp.png)

恭喜，已經完成 Serverless範例 :innocent:

## 觀察監控數據

OpenFaaS 使用 [Prometheus](https://prometheus.io/) 進行監控數據的採集。

使用瀏覽器，打開 http://your_host_ip:9090/

就會看到 Prometheus，切換到 Graph Tab，可以輸入條件並產生圖表

![serverless](https://github.com/likeitresource/run_serverless_in_few_minutes/blob/master/images/prometheus.png)


## 參考資料

* [Your first serverless Python function with OpenFaaS](https://blog.alexellis.io/first-faas-python-function/)
