---
title: 'CentOS服务器配置'
date: 2024-01-21T16:42:17+08:00
draft: false
---

## 1. 添加普通用户

> 通过VNC登录root账户，再进行以下操作

```bash
adduser abc
passwd abc

# 添加sudo权限
visudo
# 在"root       ALL=(ALL)       ALL"下面，添加"abc    ALL=(ALL)       ALL"
```

## 2. 安全设置（重要）

在**本地电脑**进行如下操作

```bash
# 生成SSH密钥
ssh-keygen -t ed25519 -f ~/.ssh/server
# 将公钥传到服务器上，本地的私钥要保存好！
ssh-copy-id -i ~/.ssh/server/server.pub -p 22 abc@123.123.123.123
```

在**云服务器**进行如下操作

> 这里要通过VNC登录，不要用ssh，否则启动防火墙后ssh会断开连接

```bash
# 修改SSH端口
sudo sed -i 's/#Port 22/Port 11022/' /etc/ssh/sshd_config
# 禁止root登录; 关闭密码登录（密钥登录更加安全）
sudo tee -a /etc/ssh/sshd_config <<-'EOF'
PermitRootLogin no
UseDNS no
PasswordAuthentication no
EOF
# 重启SSH服务
sudo systemctl restart sshd

# 启动防火墙
sudo systemctl start firewalld.service
sudo systemctl enable firewalld.service
# 允许SSH端口通过防火墙
sudo firewall-cmd --zone=public --add-port=11022/tcp --permanent
sudo systemctl restart firewalld.service
```

## 3. 操作日志（可选）

> 倒数第2行的“EOF”是结束符号

```bash
sudo tee -a /etc/profile <<-'EOF'
#user operation history
history
USER=`whoami`
USER_IP=`who -u am i 2>/dev/null| awk '{print $NF}'|sed -e 's/[()]//g'`
if [ "$USER_IP" = "" ]; then
    USER_IP=`hostname`
fi
if [ ! -d /var/log/history ]; then
    mkdir /var/log/history
    chmod 777 /var/log/history
fi
if [ ! -d /var/log/history/${LOGNAME} ]; then
    mkdir /var/log/history/${LOGNAME}
    chown -R ${LOGNAME}:${LOGNAME} /var/log/history/${LOGNAME}
    chmod 770 /var/log/history/${LOGNAME}
fi
export HISTSIZE=4096
DT=`date +"%Y%m%d_%H:%M:%S"`
export HISTFILE="/var/log/history/${LOGNAME}/${USER}@${USER_IP}_$DT"
chmod 660 /var/log/history/${LOGNAME}/*history* 2>/dev/null
EOF

source /etc/profile
```

## 4. 工具

### 系统工具

```bash
sudo yum update
sudo yum install net-tools vim wget curl nc gcc gcc-c++
```

### Docker

> 同时会安装docker compose插件

```bash
# 卸载旧版本
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
# 安装
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
# 启动
sudo systemctl start docker
sudo systemctl enable docker
# 镜像加速
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": [
      "https://mirror.ccs.tencentyun.com",
      "https://docker.m.daocloud.io",
      "https://hub-mirror.c.163.com",
      "https://dockerproxy.com",
      "https://mirror.baidubce.com",
      "https://docker.nju.edu.cn",
      "https://docker.mirrors.sjtug.sjtu.edu.cn"
  ]
}
EOF
# 添加用户组
sudo usermod -aG docker ${USER}
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### Git

> 通过yum安装的git版本比较老，但在服务器上够用了

```bash
sudo yum install git
git config --global user.name "abc"
git config --global user.email "abc@163.com"
```

### Maven

> 以maven 3.8.1版本为例

```bash
cd ~
wget https://archive.apache.org/dist/maven/maven-3/3.8.1/binaries/apache-maven-3.8.1-bin.tar.gz
tar -zxvf apache-maven-3.8.1-bin.tar.gz
sudo mv apache-maven-3.8.1 /opt/
# 配置环境变量
sudo tee -a /etc/profile <<-'EOF'
export PATH=/opt/apache-maven-3.8.1/bin:$PATH
EOF
source /etc/profile
# 镜像加速
sudo vi /opt/apache-maven-3.8.1/conf/settings.xml
# 添加如下节点
    <mirror>
      <id>aliyunmaven</id>
      <mirrorOf>*</mirrorOf>
      <name>阿里云公共仓库</name>
      <url>https://maven.aliyun.com/repository/public</url>
    </mirror>
# 复制到用户主目录
mkdir ~/.m2
cd .m2
sudo cp /opt/apache-maven-3.8.1/conf/settings.xml ~/.m2/
# 修改文件权限
sudo chown abc:abc settings.xml
```

## 5. 运行环境

### JRE 1.8

```bash
sudo yum install java-1.8.0-openjdk
sudo tee -a ~/.bashrc <<-'EOF'
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.392.b08-2.el7_9.x86_64
EOF
```

### Node 16（通过nvm安装）

先去github查看nvm最新版本号，这里以0.39.7为例

```bash
cd ~
# 安装nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
# 安装node 16
nvm install 16
# 配置淘宝镜像源
npm config set registry https://registry.npm.taobao.org
```

> 似乎centos内核版本太低？不支持node16以上的版本

### Python 3

> Linux一般自带python，下面仅安装Python虚拟环境工具virtualenv

```bash
# 安装virtualenv
sudo pip3 install virtualenv -i https://pypi.tuna.tsinghua.edu.cn/simple
# 配置镜像加速
sudo mkdir -p ~/.pip
sudo tee ~/.pip/pip.conf <<-'EOF'
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
[install]
trusted-host = https://pypi.tuna.tsinghua.edu.cn
EOF
```

## 6. 数据库

### MySQL 5.7

```bash
sudo rpm -ivh https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022
sudo yum install mysql-community-server
sudo systemctl start mysqld
sudo systemctl enable mysqld

# 获取临时密码，复制
sudo cat /var/log/mysqld.log | grep -i 'temporary password'
# 改root密码（8位以上，含大小写字母、数字、特殊字符），其他一律按y即可
mysql_secure_installation
# 添加新用户
mysql -uroot -p
create user 'abc'@'%' identified with mysql_native_password by 'g4~XN_sD^%5';
grant all privileges on *.* to 'abc'@'%';
flush privileges;
```

### Redis

```bash
cd ~
wget https://download.redis.io/redis-stable.tar.gz
tar -zxvf redis-stable.tar.gz
cd redis-stable
make
sudo make install
make clean

cd utils/
vi ./install_server.sh
# 注释以下几行代码后，再次执行./install_server.sh：

#bail if this system is managed by systemd
#_pid_1_exe="$(readlink -f /proc/1/exe)"
#if [ "${_pid_1_exe#*/}" = systemd ]
#then
#       echo "This systems seems to use systemd."
#       echo "Please take a look at the provided example service unit files in this directory, and adapt and install them. Sorry!"
#       exit 1
#fi

sudo ./install_server.sh # 全部选择默认即可，由于executable path没有默认项，故输入/usr/local/bin/redis-server

# 添加软链接
sudo ln -s /usr/local/bin/redis-cli /usr/bin/redis-cli

# （可选）添加密码(最好不要有符号，容易出bug)
sudo sed -i 's/# requirepass foobared/requirepass StizW2HMQe/' /etc/redis/6379.conf
# （可选）修改重启命令，否则会因为没有密码而阻塞
sudo sed -i 's/$CLIEXEC -p $REDISPORT shutdown/$CLIEXEC -p $REDISPORT -a "StizW2HMQe" shutdown/' /etc/rc.d/init.d/redis_6379

sudo systemctl daemon-reload
sudo systemctl restart redis_6379.service
```

### MongoDB

> 待完善，慎用

```bash
sudo tee /etc/yum.repos.d/mongodb-org-6.0.repo <<-'EOF'
[mongodb-org-6.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/6.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-6.0.asc
EOF

sudo yum install mongodb-org
sudo systemctl daemon-reload
sudo systemctl start mongod
sudo systemctl enable mongod

# 进入mongo shell
mongosh

# 创建超级管理员
use admin
db.createUser({
    user:"root",
    pwd:"2YaM&5h&SAf%MHT",
    roles:[{role:"userAdminAnyDatabase",db:"admin"}]
})
# 创建业务数据库管理员
db.createUser({
    user:"abc",
    pwd:"TaTYhCr3",
    roles:[{role:"readWrite",db:"test"}]
})
exit

# 开启密码鉴权
sudo sed -i 's/#security:/security:\n    authorization: enabled/' /etc/mongod.conf
sudo systemctl restart mongod
```

## 7. Web服务

### Tomcat 9

> 以tomcat 9.0.78为例

```bash
cd ~
wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.78/bin/apache-tomcat-9.0.78.tar.gz
tar -zxvf apache-tomcat-9.0.78.tar.gz
sudo mv apache-tomcat-9.0.78 /opt/
# 自启动
sudo tee /etc/systemd/system/tomcat.service <<-'EOF'
[Unit]
Description=Tomcat
After=syslog.target network.target remote-fs.target nss-lookup.target

[Service]
Type=oneshot
ExecStart=/opt/apache-tomcat-9.0.78/bin/startup.sh
ExecStop=/opt/apache-tomcat-9.0.78/bin/shutdown.sh
ExecReload=/bin/kill -s HUP $MAINPID
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl start tomcat
sudo systemctl enable tomcat
```

### Nginx

> 以nginx 1.24.0为例

```bash
sudo yum install pcre pcre-devel zlib zlib-devel openssl openssl-devel
cd ~
wget https://nginx.org/download/nginx-1.24.0.tar.gz
tar -zxvf nginx-1.24.0.tar.gz
cd nginx-1.24.0/
./configure --prefix=/opt/nginx --with-http_stub_status_module --with-http_ssl_module --with-http_gzip_static_module
make
sudo make install
make clean

# 自启动
sudo tee /etc/systemd/system/nginx.service <<-'EOF'
[Unit]
Description=nginx
After=network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
ExecStart=/opt/nginx/sbin/nginx -c /opt/nginx/conf/nginx.conf
ExecReload=/opt/nginx/sbin/nginx -s reload -c /opt/nginx/conf/nginx.conf
ExecStop=/opt/nginx/sbin/nginx -s quit
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start nginx
sudo systemctl enable nginx
# 防火墙
sudo firewall-cmd --zone=public --add-port=80/tcp --permanent
sudo firewall-cmd --zone=public --add-port=443/tcp --permanent
sudo systemctl restart firewalld.service
```
