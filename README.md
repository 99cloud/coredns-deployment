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
2. 修改 *[path]/coredns-deployment/inventory_no_k8s.example*文件

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
    ansible-playbook -i inventory_no_k8s.example deploy_all_without_k8s.yml
    ```

### 创建coredns的lb

1. 在云平台上创建lb把*节点1、节点2、节点3* 做为该lb的member