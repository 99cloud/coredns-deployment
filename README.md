# coredns-deployment

- 主要用于在边缘方案中部署整套coredns的产品。

## 准备工作

1. （可选）如果你没有 openshift 已经安装好，可以按照以下步骤完成，会在你本地启动一个 all in one 的 openshift 集群:

        yum install docker -y
        echo -e "{ \"insecure-registries\": [\"172.30.0.0/16\"] }" > /etc/docker/daemon.json
        systemctl start docker
        wget https://github.com/openshift/origin/releases/download/v3.11.0/openshift-origin-client-tools-v3.11.0-0cbc58b-linux-64bit.tar.gz -O oc.tar.gz
        tar -zxvf oc.tar.gz --strip-components=1 -C /usr/bin 
        oc cluster up

1. （可选）如果你选择用 ansible 推本地的机器，请保证你安装的ansible：

        yum install ansible -y
        sed -i '/\[defaults\]/a host_key_checking = False' /etc/ansible/ansible.cfg
        yum update -y

## 开始部署

1. 下载项目的repo:

        git clone http://172.16.30.21/openshift_origin/coredns-edge-deployment.git

1. 修改 inventory.example 文件中的 *master1* 为你适合你的主机
1. 如果你不希望 *etcd* 数据使用tls的认证方式的话，请修改 inventory.example 中的 `etcd_use_tls` 为 *no*
1. 执行ansible:

        cd [path]/coredns-edge-deployment
        ansible-playbook -i inventory.example deploy_all.yml