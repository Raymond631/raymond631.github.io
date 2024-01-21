---
title: 'Linux常用命令'
date: 2024-01-21T16:43:31+08:00
draft: false
---

## 防火墙

### Ubuntu

```bash
# 开启防火墙
sudo ufw enable

# 开放端口
sudo ufw allow 8080/tcp
sudo ufw reload

# 关闭端口
sudo ufw delete allow 8080/tcp
sudo ufw reload

# 查看端口
sudo ufw status
```

### CentOS

```bash
# 开启防火墙
sudo systemctl start firewalld.service
sudo systemctl enable firewalld.service

# 开放端口
sudo firewall-cmd --zone=public --add-port=3306/tcp --permanent
sudo firewall-cmd --reload

# 关闭端口
sudo firewall-cmd --remove-port=3306/tcp --permanent
sudo firewall-cmd --reload

# 查看端口
sudo firewall-cmd --list-ports
```



## 软件安装与卸载（4种方法）

### 源码安装

```shell
# 安装
tar -zxvf test.tar.gz
cd test/
./configure --prefix=/opt/test
make
sudo make install
sudo rm -r test/

# 卸载(必须先停掉相关进程！)
sudo rm -r /opt/test
```

### 预编译包

```shell
# 安装
tar -zxvf test.tar.gz
sudo mv test /opt/test

# 配置环境变量
sudo tee -a ~/.bashrc <<-'EOF'
export PATH=/opt/test/bin:$PATH
EOF
source ~/.bashrc

# 卸载(需要删除环境变量)
sudo rm -r /opt/test
```

### deb、rpm安装包

```shell
# 安装
sudo dpkg -i tset.deb
sudo rpm -ivh test.deb

# 查找已安装软件，可搭配grep使用
sudo dpkg -l
sudo rpm -qa

# 卸载
sudo dpkg -r test
sudo rpm -e test
```

### apt、yum在线安装

```bash
# 在线查找
apt-cache search package_name
yum search package_name

# 安装
sudo apt install -y test
sudo yum install -y test

# 卸载
sudo apt purge test
sudo yum remove test
```



## 查看系统状态

```bash
# 内存使用率
free | awk 'NR==2{print ($2-$7)/$2*100"%"}'

# 剩余硬盘容量
df -h | grep /dev/vda1|awk '{print $4}'

# CPU负载（应该<0.7）
uptime | awk '{print $12}'

# 进程列表
top
```



## Docker

```bash
# 所有镜像整体导出
docker compose images | awk 'FNR > 1 {print $2":"$3}'| sort -u | xargs docker save -o images.tar

# 进入容器内部
docker exec -it container_name bash

# 将容器内的文件复制到宿主机（也可以反过来用）
docker cp container_id:path_in_container path_in_host
 
# 启动容器
sudo docker run -d -p 80:80 -v /opt/nginx/conf:/etc/nginx --name nginx nginx:latest
```



## 文件操作

```bash
# 从目录下所有文件内容中查找（pattern为正则表达式）
grep -r -n pattern dir/

# 替换单行文本（old_str换成new_str）
sed -i 's/old_str/new_str/g' test.txt

# 追加多行文本
tee -a test.txt <<-'EOF'
this is the first line.
this is the second line.
EOF

# 搜索文件(-i 忽略大小写; * 通配符)
find ./ -name -i "*file.txt"
```



## 其他命令

```bash
# 添加后台任务
nohup your_command >> log_path 2>&1 &
# 删除后台任务
ps -ef | grep xxx
kill -9 进程号

# 抓取整站
wget -r  -p -np -k -E https://www.example.com

# unzip中文乱码 解决办法
unzip -O cp936 xxx.zip

# 刷新DNS缓存
sudo systemctl restart systemd-resolved

# 切换JDK版本
sudo update-alternatives --config java
```



## Bash常用快捷键

* ctrl a：跳到行首

* ctrl e：跳到行尾

* ctrl u：剪切整行

* ctrl y：粘贴最近剪切的文本

* ctrl l：相当于clear

* tab：补全

* ctrl c：终止

* ctrl d：退出

* ctrl insert 或 ctrl shift c：复制

* shift insert 或 ctrl shift v：粘贴
