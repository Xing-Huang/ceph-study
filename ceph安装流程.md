
# **软件版本**
|软件名称|版本|
|:-|:-:|
|OS| CentOS Linux release 7.9.2009|
|Kernel| 3.10.0-693.el7.x86_64|
|ceph| Luminous 12.2.13|
|ceph-deploy| 2.0.1 |


# **集群环境规划**
| 集群 | Cluster Network | Public Network|Services|
|:-|:-:|:-:|:-:|
| controller| 172.22.92.10| 172.22.94.10|ceph-deploy|
| ceph-1| 172.22.92.11| 172.22.94.11|osd.0, mon.ceph-1|
| ceph-2| 172.22.92.12| 172.22.94.12|osd.1|
| ceph-3| 172.22.92.13| 172.22.94.13|osd.2|
| client| 172.22.92.14| 172.22.94.14|client|


# **配置部署环境**

* ## **关闭防火墙**  
  关闭本地防火墙，在所有节点执行:
  ```sh
  systemctl stop firewalld
  systemctl disable firewalld
  systemctl status firewalld
  ```
* ## 关闭selinux
  关闭selinux，在所有节点执行:
  ```sh
  setenforce 0
  sed -i '7s/enforcing/disabled/' /etc/sysconfig/selinux
  ```
    
* ## **配置主机名**  
  配置主机名，配置分别为controller，ceph-1，ceph-2，ceph-3， client,在各自主机执行:
  ```sh
  hostnamectl set-hostname controller
  hostnamectl set-hostname ceph-1
  hostnamectl set-hostname ceph-2
  hostnamectl set-hostname ceph-3
  hostnamectl set-hostname client
  ```
* ## 修改域名解析文件
  修改域名解析文件，所有节点执行：
  ```sh
  echo "\
  172.22.92.10  controller\
  172.22.92.11  ceph-1\
  172.22.92.12  ceph-2\
  172.22.92.13  ceph-3\
  " >> /etc/hosts
  ```
* ## **配置ntp**
  配置ntp，ceph-1为ntp服务主节点，在ceph-1节点执行:
  ```sh
  yum -y install chrony
  timedatectl set-timezone Asia/Shanghai
  echo -e "allow 172.22.92.0/24\\nlocal stratum 10" >> /etc/chrony.conf
  systemctl enable chronyd.service
  systemctl restart chronyd.service
  chronyc sources
  ```
  配置ceph-2，ceph-3为从节点，ceph-2, ceph-3节点执行:
  ```sh
  yum -y install chrony
  timedatectl set-timezone Asia/Shanghai
  echo "server controller iburst" >> /etc/chrony.conf
  sed -i "3,6s/^/#/" /etc/chrony.conf
  systemctl enable chronyd.service
  systemctl restart chronyd.service
  chronyc sources
  ```

* ## **配置免密登录**
  设置controller节点免密登录ceph节点，在controller节点执行:
  ```sh
  ssh-keygen -t rsa
  ssh-copy-id root@ceph-1
  ssh-copy-id root@ceph-2
  ssh-copy-id root@ceph-3
  ```
* ## **配置ceph镜像源**
  配置清华镜像源,在所有节点执行:
  ```sh
  rpm -ivh https://mirrors.tuna.tsinghua.edu.cn/ceph/rpm-luminous/el7/noarch/ceph-release-1-1.el7.noarch.rpm
  ```

# **安装ceph**
* ## **安装ceph软件**
    安装ceph和ceph-radosgw，在所有节点上执行:
    ```sh
    sudo yum install -y epel-release 
    sudo yum install -y ceph ceph-radosgw
    ```
    在各节点查看版本:
    ```sh
    ceph -v 
    ```
    获取的版本信息:
    ```
    ceph version 12.2.13 (584a20eb0237c657dc0567da126be145106aa47e) luminous (stable)
    ```
    安装ceph-deploy，在deploy节点上执行:
    ```sh
    sudo yum install -y ceph-deploy
    ```
* ## **部署mon节点**
    **以下指令在deploy节点执行**  
    创建集群，在ceph-1、ceph-2、ceph-3上添加mon:
    ```sh
    mkdir cluster && cd cluster
    ceph-deploy new --cluster-network=172.22.92.0/24 --public-network=172.22.94.0/24 ceph-1 ceph-2 ceph-3
    ```

    修改ceph.conf配置文件，允许删除pool，在最后添加如下配置:
    ```sh
    echo -e "[mon]\n\
    mon_allow_pool_delete = true" >> ceph.conf
    ```

    初始化监视器并收集密钥：
    ```sh
    ceph-deploy mon create-initial
    ```

    推送"ceph.conf"和"ceph.client.admin.keyring"拷贝到所有节点:
    ```sh
    ceph-deploy --overwrite-conf admin controller ceph-1 ceph-2 ceph-3 client
    ```


    查看配置是否成功:
    ```sh
    ceph -s
    ```
    获取的信息:
    ```
    ```
* ## **部署mgr节点**
    **以下指令在deploy节点执行**
    部署mgr节点：
    ```sh
    ceph-deploy mgr create ceph-1 ceph-2 ceph-3
    ```
    查看MGR是否部署成功：
    ```sh
    ceph -s
    ```
* ## **部署osd节点**
    查看各节点可用的硬盘，在各个ceph节点执行:
    ```sh
    lsblk
    ```
    创建osd节点，在deploy节点执行:
    ```sh
    ceph-deploy osd create ceph-1 --data /dev/vdb
    ceph-deploy osd create ceph-2 --data /dev/vdb
    ceph-deploy osd create ceph-3 --data /dev/vdb
    ```
# 验证ceph
* **创建存储池**  
    **以下所有指令在client节点执行**
    创建prbd存储池, pgnum为128：
    ```sh
    ceph osd pool create prbd 128 128
    ```
    * pgnum = (OSD数量 * 100)/(数据冗余因数*poolnum)。  
    * 数据冗余因数对副本模式而言是副本数，对EC模式而言是数据块+校验块之和。  
    * pgnum最好是2的整数幂
  
    (可选)Ceph 14.2.10版本创建存储池后，需指定池类型（CephFS、RBD、RGW）三种，本文以创建块存储为例，在client节点执行:
    ```sh
    ceph osd pool application enable prbd rbd
    ```

* **创建块设备**
    **以下所有指令在client节点执行**
    创建块设备：
    ```sh
    rbd create image1 --size 10G --pool prbd --image-feature layering
    ```
    查看块设备:
    ```sh
    rbd ls -p prbd
    ```
* **映射块设备镜像**
    **以下所有指令在client节点执行**
    映射块设备image1
    ```sh
    rbd map prbd/image1
    ```
    
    




