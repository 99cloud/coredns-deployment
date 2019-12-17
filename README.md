# coredns-deployment

- 主要用于在边缘方案中部署整套coredns的产品。

## 非k8s解决方案的部署

1. 在非k8s集群里部署*coredns*、*coredns-speaker(api)*、*消息队列nats(可选)*、*etcd数据库集群*

## 软件架构

    ```console
    # 下发规则(no nats)
                                   |---- coredns api节点1 <--->etcd节点1、etcd节点2、etcd节点3(内置lb)|                                          |------coredns节点1
    mep ----> coredns api lb <---->|---- coredns api节点2 <--->etcd节点1、etcd节点2、etcd节点3(内置lb)| <-----> coredns api lb <--从api获取数据--->|------coredns节点2
                                   |---- coredns api节点3 <--->etcd节点1、etcd节点2、etcd节点3(内置lb)|                                          |------coredns节点3

    # 使用dns
                                    |---- coredns节点1 ----|
    解析域名请求 ---> coredns lb <--->|---- coredns节点2 ----|
                                    |---- coredns节点3 ----|                           
    ```

## 文档使用注意事项

1. 请不要漏掉文档流程的中任何一个步骤，以免不必要的错误
2. 本项目并不是完全离线安装，可以说90%都是离线的，但有些还是依赖internet

### 部署架构

1. 以下是在非k8s环境下部署coredns edge的解决方案，我们为了方便都合并到一台机器中去:

|  | coredns | coredns api | nats(消息队列) | etcd（数据库） | ansible |
| --- | --- | --- | --- | --- | --- |
| 部署节点 | No | No | No | No | Yes |
| 节点1 | Yes | Yes | Optional | Yes | No |
| 节点2 | Yes | Yes | Optional | Yes | No |
| 节点3 | Yes | Yes | Optional | Yes | No |

2. 文档中默认的测试走通的环境配置如下:

|  | ip | hostname | os | iso | 运行环境 |
| --- | --- | --- | --- | --- | --- |
| 部署节点 | 10.0.0.28 | etcd-node1 | centos7-1810 | CentOS-7-x86_64-Minimal-1810.iso | kvm guest |
| 节点1 | 10.0.0.28 | etcd-node1 | centos7-1810 | CentOS-7-x86_64-Minimal-1810.iso | kvm guest |
| 节点2 | 10.0.0.29 | etcd-node2 | centos7-1810 | CentOS-7-x86_64-Minimal-1810.iso | kvm guest |
| 节点3 | 10.0.0.30 | etcd-nope3 | centos7-1810 | CentOS-7-x86_64-Minimal-1810.iso | kvm guest |

3. 文档中默认的测试走通的硬件配置如下:

|  | ip | hostname | cpu | memory | disk | 运行环境 |
| --- | --- | --- | --- | --- | --- | --- |
| 节点1/部署节点 | 10.0.0.28 | etcd-node1 | 4 | 16Gi | 250Gi | kvm guest |
| 节点2 | 10.0.0.29 | etcd-node2 | 4 | 16Gi | 250Gi | kvm guest |
| 节点3 | 10.0.0.30 | etcd-nope3 | 4 | 16Gi | 250Gi | kvm guest |

4. 服务在主机上承载的模式

|  | coredns | coredns api | nats(消息队列) | etcd（数据库） |
| --- | --- | --- | --- | --- |
| 节点1 | systemd | container + systemd | N/A | systemd |
| 节点2 | systemd | container + systemd | N/A | systemd |
| 节点3 | systemd | container + systemd | N/A | systemd |

5. 所有组件的systemd名称

|  | coredns | coredns api | nats(消息队列) | etcd（数据库） |
| --- | --- | --- | --- | --- |
| systemd名称 | coredns | coredns-api | N/A | etcd |

6. 不在脚本中部署且需要自己部署的的组件Load balancer:

|  | coredns | coredns api | nats(消息队列) | etcd（数据库） |
| --- | --- | --- | --- | --- |
| Load balancer | Yes | Yes | Optional | No |

7. Load balancer的配置:

|  | member port | protocal |
| --- | --- | --- |
| coredns lb | {{ coredns_service_port }}(根据参数*coredns_service_port*，默认53) | UDP |
| coredns api lb | {{ coredns_api_port }}(根据参数*coredns_api_port*，默认80) | TCP |

8. Load balancer的配置:

    ```console
                    |----- 节点1:{{ coredns_api_port }}
    coredns api lb--|----- 节点2:{{ coredns_api_port }}
                    |----- 节点3:{{ coredns_api_port }}

                    |----- 节点1:{{ coredns_service_port }}
    coredns lb------|----- 节点2:{{ coredns_service_port }}
                    |----- 节点3:{{ coredns_service_port }}

    ```

### 准备工作

1. (可选)在没有dns的环境下部署，我们需要修改被部署的三台主机*节点1、节点2、节点3的 */etc/hosts* 文件

    ```console
    # 默认配置
    10.0.0.28	etcd-node1
    10.0.0.29	etcd-node2
    10.0.0.30	etcd-node3
    ```

2. 关闭三台部署主机*selinux*,修改*/etc/selinux/config*为*SELINUX=disabled*
3. 重启这些主机让关闭selinux操作生效
4. 登陆到部署节点*节点1/部署节点*
5. 在*部署节点上*下载repo

    ```console
    $ yum install git -y
    $ git clone https://user:password@github.com/99cloud/coredns-deployment.git
    ```

6. 在*部署节点上*确认所有的节点通过主机名可以被访问

    ```console
    $ ping etcd-node1 # 解析10.0.0.28
    $ ping etcd-node2 # 解析10.0.0.29
    $ ping etcd-node3 # 解析10.0.0.30
    ```

7. (可选)确保三台主机的*53*端口和*80*端口都没被占用
8. 创建外部的lb给到*coredns api*作为负载均衡

### 开始部署

1. 在*部署节点上*进入项目目录*coredns-deployment*
2. 修改 *[path]/coredns-deployment/inventory_k8s.example*文件中那些不符合你需求的配置

    ```console
    etcd_node1_ip="10.0.0.28" # 如果修改了ip，只需要改这里
    etcd_node2_ip="10.0.0.29" # 如果修改了ip，只需要改这里
    etcd_node3_ip="10.0.0.30" # 如果修改了ip，只需要改这里
    coredns_upstream_address=114.114.114.114 # 那些不被分流的域名通过这个上游dns地址来正常解析
    coredns_service_port=53 # coredns运行的端口
    # 填写coredns api lb的lb的地址或者hostname
    # 比如coredns api lb的地址为172.16.100.100:8080
    # coredns_speaker_service_address="172.16.100.100:8080"
    # 这里为了方便我只写了其中的一个节点的地址
    coredns_speaker_service_address="etcd-node1"
    coredns_api_port=80 #coredns api提供服务的端口
    
    # 不要改，逻辑是在本地先签名证书，所以你运行ansible的时候可以是在一个部署节点上
    # 也可以是在任何一台部署的目标机器上
    # 部署节点，在例子中就是 节点1
    [tls]
    localhost              ansible_connection=local
    
    # 请使用hostname，不要用ip因为复制证书的时候需要
    [etcd]
    etcd-node1
    etcd-node2
    etcd-node3
    #localhost              ansible_connection=local
    # 172.16.60.17  ansible_user=root ansible_ssh_private_key_file=~/.ssh/id_rsa_rhel
    
    # 可以忽略现在没有完成nats的tls部署
    [nats]
    etcd-node1
    etcd-node2
    etcd-node3
    #localhost              ansible_connection=local
    # 172.16.60.17  ansible_user=root ansible_ssh_private_key_file=~/.ssh/id_rsa_rhel
    
    [coredns_api]
    etcd-node1
    etcd-node2
    etcd-node3
    #localhost              ansible_connection=local
    # 172.16.60.17  ansible_user=root ansible_ssh_private_key_file=~/.ssh/id_rsa_rhel
    
    [coredns]
    etcd-node1
    etcd-node2
    etcd-node3
    #localhost              ansible_connection=local
    # 172.16.60.17  ansible_user=root ansible_ssh_private_key_file=~/.ssh/id_rsa_rhel
    ```

3. 在*部署节点上*运行命令

    ```console
    $ ansible-playbook -i inventory_no_k8s.example deploy_all_without_k8s.yml
    ```

### 创建coredns的lb

1. 在云平台上创建lb把*节点1、节点2、节点3* 做为该lb的member

### 检验安装

1. 检验coredns api运行正常

    ```console
    # 检查节点1
    # 命令应该返回结果 { "mgs": "OK" }
    $ curl -X PUT http://etcd-node1/99cloud/coredns-speaker/1.0.0/hijack\?token\=token1 /
    -d "{\"id\":\"123124\",\"action\":\"create\",\"domain\":\"www.ccc.com\",\"answers\":[{\"domain\":\"www.ccc.com\",\"type\":\"A\",\"ip\":\"12.12.12.12\"}]}"

    # 检查节点2
    # 命令应该返回结果 { "mgs": "OK" }
    $ ^etcd-node1^etcd-node2^

    # 检查节点3
    # 命令应该返回结果 { "mgs": "OK" }
    $ ^etcd-node2^etcd-node3^

    # load balancer
    # 命令应该返回结果 { "mgs": "OK" }
    $ ^etcd-node3^load balance的地址^
    ```

1. 检验coredns运行正常

    ```console
    # 检查节点1
    $ dig www.ccc.com @10.0.0.28

    # 检查节点2
    $ dig www.ccc.com @10.0.0.29

    # 检查节点3
    $ dig www.ccc.com @10.0.0.30

    # load balancer
    $ dig www.ccc.com @[load balancer地址]

    # 返回结果
    # ... dig相关信息
    ;; OPT PSEUDOSECTION:
    ; EDNS: version: 0, flags:; udp: 4096
    ; COOKIE: 7a71203fbcaa2394 (echoed)
    ;; QUESTION SECTION:
    ;www.ccc.com.			IN	A

    ;; ANSWER SECTION:
    www.ccc.com.		0	IN	A	12.12.12.12

    ;; ADDITIONAL SECTION:
    _udp.www.ccc.com.	0	IN	SRV	0 0 59460 .
    # ... dns server信息
    ```

### Trouble shooting

1. 安装中出现*docker pull image timeout*，下载我预先制作好的镜像*99cloud/coredns-management-api*失败了

    > 检查网络是否可以到外网

    > 检查dns域名是否可以解析到docker.io

2. curl的时候出现*deadline exceed错误*，数据库连接超时经常出现的问题，可以做以下检查定位错误

    ```console
    # 节点1、2、3中的任意一个节点运行命令
    # 进入容器
    $ docker exec -it coredns-speaker /bin/sh
    容器:/> cd /etc/coredns-edge-deployment/etcd/certs
    容器:/> etcdctl --cacert=etcd-client-ca.crt --cert=etcd-client.crt --key=etcd-client.key --endpoints=https://[你的etcd的数据库的地址一般有3个]:2379 member list -w table --debug
    # 以上命令会打印debug的日志，就会看到问题输出了
    ```

3. dig的时候没有将我们分流的域名给出正确的应答

    > 我们使用的coredns的ASK模式，这个模式是定期（10秒）去问coredns api获取最新数据，所以请等待10秒再试一下

    > 如果过了时间还是没有返回正确结果检查容器日志 *docker logs coredns-speaker -f*

## 在k8s解决方案的部署

1. 在非k8s集群里部署*coredns*、*coredns-speaker(api)*、*消息队列nats(可选)*、*etcd数据库集群*

## 软件架构

    ```console
    # 下发规则(no nats)
                                   |--- coredns api pod1 <---> |                                                                 |------coredns pod1
    mep ---> coredns api svc <---->|--- coredns api pod2 <---> |<--->etcd peer service| <----> coredns api svc <--从api获取数据--->|------coredns pod2
                                   |--- coredns api pod3 <---> |                                                                 |------coredns pod3

    # 下发规则(with nats)
                                   |--- coredns api pod1 <---> |                                                        |------coredns pod1
    mep ---> coredns api svc <---->|--- coredns api pod2 <---> | ---> message queue(nats) <--- 从nats消息队列里获取数据 --->|------coredns pod2
                                   |--- coredns api pod3 <---> |                                                        |------coredns pod3           

    # 使用dns
                                     |---- coredns pod1 ----|
    解析域名请求 ---> coredns svc <--->|---- coredns pod2 ----|
                                     |---- coredns pod3 ----|                           
    ```

## 文档使用注意事项

1. 请不要漏掉文档流程的中任何一个步骤，以免不必要的错误
2. 本项目并不是完全离线安装，可以说90%都是离线的，但有些还是依赖internet，可以通过预先下载镜像和制作虚拟机镜像来解决

    > image: 99cloud/coredns-management-api:latest

    > image: 99cloud/coredns-edge:latest

    > software: docker

    > software: git(可选如果你预先复制了repo)

### 部署架构

1. 取决于k8s的架构

### 准备工作

1. (可选)如果是在openshift平台上运行请使用命令`oc edit oc edit scc restricted`，并修改 `runAsUser: RunAsAny`
2. 在*部署节点上*下载repo

    ```console
    $ yum install git,ansible -y
    $ git clone https://user:password@github.com/99cloud/coredns-deployment.git
    ```
3. 请确保kubectl已经正确配置为管理身份

### 开始部署

1. 在*部署节点上*进入项目目录*coredns-deployment*
2. 修改 *[path]/coredns-deployment/inventory_k8s.example*文件中那些不符合你需求的配置

    ```console
    [all:vars]
    # basic section starts
    oc_binary="/usr/bin/kubectl"
    namespace_coredns_edge="coredns-edge"
    caas_cluster_domain="cluster.local"
    # 当选择passive模式的话，将跳过nats消息队列的部署
    coredns_speaker_mode="initiative"
    # basic section ends

    # tls sign section starts
    # 证书签名按需修改
    tls_c="CHINA"
    tls_l="Shanghai"
    tls_o="99cloud"
    tls_st="Shanghai"
    tls_ou="CAAS"
    # tls sign section ends

    # etcd operator section starts
    # 默认就好了
    etcd_peer_svc_name="etcd-coredns-speaker"
    # 不要修用tls安全
    etcd_use_tls="yes"
    # 修改为k8s集群山现有的storage class
    etcd_storage_class_name="alicloud-nas"
    # etcd数据库的磁盘大小
    etcd_storage_size=30
    # 如果使用tls请忽略
    etcd_username=username
    etcd_password=password
    # etcd operator section ends

    # nats section starts
    # 使用tls
    nats_use_tls="yes"
    nats_cluster_name="nats-cluster"
    # 消息队列的频道
    message_queue_subject="msg.edge.coredns.99cloud"
    # 如果用tls请忽略
    message_queue_username="username"
    message_queue_password="password"
    # nats section ends

    # coredns management api section starts
    # 如果数据库是tls必须是https
    database_connection_protocol="https"
    api_auth_mode="token" # 目前只支持token模式认证
    allowed_token="token1 | token2 | token3"
    # coredns management api section ends

    # coredns message plugin section starts
    coredns_cluster_name=coredns
    coredns_service_port=53
    coredns_upstream_address=114.114.114.114
    coredns_speaker_service_address="{{ 'coredns-speaker-svc.' + namespace_coredns_edge + '.svc' }}"
    # 注意token必须是上面"allowed_token"中的一个
    initial_data_provider_service_url="{{ coredns_speaker_service_address + '/99cloud/coredns-speaker/1.0.0/hijack-list?token=token1' }}"
    # 建议不小10秒
    # 如果设置coredns_speaker_mode="initiative"请忽略
    ask_frequency=10
    # 静态pod的目录地址
    pod_manifest_path="/etc/kubernetes/manifests"
    # 不要修改除非你重新制作镜像
    conatiner_config_dir_path="/etc/coredns/"
    # coredns message plugin section ends

    # k8s 任何一个master节点
    [master1]
    localhost              ansible_connection=local
    # 172.16.60.17  ansible_user=root ansible_ssh_private_key_file=~/.ssh/id_rsa_rhel

    # 所有你想要部署的coredns的节点
    # 注意这些节点必须是受k8s管理的节点
    [dns]
    localhost              ansible_connection=local
    # 172.16.60.17  ansible_user=root ansible_ssh_private_key_file=~/.ssh/id_rsa_rhel
    # 172.16.60.18  ansible_user=root ansible_ssh_private_key_file=~/.ssh/id_rsa_rhel
    # 172.16.60.19  ansible_user=root ansible_ssh_private_key_file=~/.ssh/id_rsa_rhel

    # 不要改，逻辑是在本地先签名证书，所以你运行ansible的时候可以是在一个部署节点上
    # 也可以是在任何一台部署的目标机器上或者和[master1]的机器中的任何一台上
    # 因为部署dns的时候是static pod，而且部署dns的机器不一定是master节点
    [tls]
    localhost              ansible_connection=local                       
    ```

3. 运行命令 `ansible-playbook -i inventory_k8s.example deploy_all_with_k8s.yml`