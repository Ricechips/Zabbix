# Zabbix

## 环境
Zabbix server:Centos7 192.168.106.201<br>
被监控端:Win10 192.168.106.3  Centos 192.168.106.2

## 监控服务端
>脚本
```c
#!/bin/bash

#安装zabbix源、aliyun YUM源
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
curl -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
rpm -ivh http://repo.zabbix.com/zabbix/3.0/rhel/7/x86_64/zabbix-release-3.0-1.el7.noarch.rpm

#安装zabbix 
yum install -y zabbix-server-mysql zabbix-web-mysql

#安装启动 mariadb数据库
yum install -y  mariadb-server
systemctl start mariadb.service

#创建数据库
mysql -e 'create database zabbix character set utf8 collate utf8_bin;'
mysql -e 'grant all privileges on zabbix.* to zabbix@localhost identified by "zabbix";'

#导入数据
zcat /usr/share/doc/zabbix-server-mysql-3.0.13/create.sql.gz|mysql -uzabbix -pzabbix zabbix

#配置zabbixserver连接mysql
sed -i.ori '115a DBPassword=zabbix' /etc/zabbix/zabbix_server.conf

#添加时区
sed -i.ori '18a php_value date.timezone  Asia/Shanghai' /etc/httpd/conf.d/zabbix.conf

#解决中文乱码
yum -y install wqy-microhei-fonts
\cp /usr/share/fonts/wqy-microhei/wqy-microhei.ttc /usr/share/fonts/dejavu/DejaVuSans.ttf

#启动服务
systemctl start zabbix-server
systemctl start httpd

#写入开机自启动
chmod +x /etc/rc.d/rc.local
cat >>/etc/rc.d/rc.local<<EOF
systemctl start mariadb.service
systemctl start httpd
systemctl start zabbix-server
EOF

#输出信息
echo "浏览器访问 http://`hostname -I|awk '{print $1}'`/zabbix"
```
> 若80端口被占用 修改*/etc/httpd/conf/httpd.conf*
> 注意SElinux和firewalld关闭状态

## 被监控端Win10
> 打通虚拟机和主机的网络:route add 192.168.106.0 mask 255.255.255.0 192.168.106.1
> 官网[跳转](https://www.zabbix.com/cn/download_agents)下载相应版本Zip<br>
修改conf文本
```c
LogFile=D:\zabbix_agentd.log
Server=192.168.106.201                                            //zabbix服务端的ip地址
ServerActive=192.168.106.201
Hostname=192.168.106.3                                           //windows客户机的ip地址
```
```c
D:\zabbix\bin\zabbix_agentd.exe -c D:\zabbix\conf\zabbix_agentd.conf -i  //安装
D:\zabbix\bin\zabbix_agentd.exe -c D:\zabbix\conf\zabbix_agentd.conf -s  //启动,-d是卸载
```
> 关闭防火墙
> 在服务端测试连通性*zabbix_get -s 92.168.106.3 -p 10050 -k "system.cpu.load[all,avg1]*

## 被监控端Centos
> *rpm -ivh http://repo.zabbix.com/zabbix/3.0/rhel/7/x86_64/zabbix-release-3.0-1.el7.noarch.rpm*
> *yum install zabbix-agent*
> *vi /etc/zabbix/zabbix_agentd.conf*修改
```c
Server=192.168.106.201     //安装zabbix服务端的机器的IP
ServerActive=192.168.106.201 //安装zabbix服务端的机器的IP
Hostname=Centos          //主机名
```
>  开启服务*systemctl start zabbix-agen*
