---
title: "zabbix游戏监控日志系统部署"
date: 2022-09-13T01:30:29+08:00
lastmod: 2022-09-13T01:30:29+08:00
author: ["frog"]
keywords:
-
categories:
-
tags:
-
description: ""
weight:
draft: false # 是否为草稿
comments: true
reward: true # 打赏
mermaid: true #是否开启mermaid
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示路径
cover:
    zabbix游戏监控日志系统部署: "" #图片路径例如：posts/tech/123/123.png
    caption: "" #图片底部描述
    alt: ""

relative: false
---

![](Typora_11_jiankongsystem1.svg)

利用bspTree原理对地图进行动态切割

 **<font color='red'>演示</font>**

<iframe src="//player.bilibili.com/player.html?aid=603017673&bvid=BV1VB4y1n7c9&cid=832077865&page=1" height="600" wigth="1024" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"> </iframe>

| 安装环境 | 版本  |
| -------- | ----- |
| Ubuntu   | 20.04 |
| zabbix   | 6.0   |
| mysql    | 8.0   |

 **<font color='red'>Ubuntu20.04+mysql8.0+zabbix6.0+elk+filebeat+logstash+grafana</font>**							

## 1. zabbix 6.0

### 1.1. 阿里云镜像地址

```sh
https://mirrors.aliyun.com/zabbix/zabbix/6.0/ubuntu/pool/main/z/zabbix-release/
```

### 1.2. 下载 zabbix

```shell
sudo wget https://repo.zabbix.com/zabbix/6.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.0-1+ubuntu20.04_all.deb 
sudo dpkg -i zabbix-release_6.0-1+ubuntu20.04_all.deb
sudo apt update
```

### 1.3. 安装Zabbix server，Web前端，agent

```shell
sudo apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent
```

### 1.4. 创建初始数据库

```shell
mysql -uroot -p123456
mysql> create database zabbix character set utf8mb4 collate utf8mb4_bin;
mysql> create user zabbix@`%` identified by '123456';
mysql> grant all privileges on zabbix.* to zabbix@`%`;
mysql> quit;
```

### 1.5. 导入初始架构和数据，系统将提示您输入新创建的密码[默认密码现在设置为 `123456`

```shell
zcat /usr/share/doc/zabbix-sql-scripts/mysql/server.sql.gz | mysql -uzabbix -p -h10.40.38.67 zabbix # 指定本地的IP地址，不默认就会指向本地localhost
```

**如果报`ERROR 2003 (HY000): Can't connect to MySQL server on '10.40.38.67:3306' (111)` 看第5章`mysql`操作指导，多半是因为权限和密码问题**

###  为Zabbix server配置数据库

```shell
sudo vim  /etc/zabbix/zabbix_server.conf
修改 DBPassword=123456
```

### 1.6. 启动Zabbix server和agent进程

```shell
sudo systemctl restart zabbix-server zabbix-agent apache2 grafana-server
sudo systemctl enable zabbix-server zabbix-agent apache2 grafana-server
```

### 1.7. 连接web前端[10.40.38.67 换成你的ip地址] [用谷歌浏览器或者microsoft Edge浏览器打开]

```shell
 http://10.40.38.67/zabbix
 默认的用户名是Admin(A是大写)，Password：zabbix
```

### 1.8. 修改时区

```sh
sudo vi /etc/apache2/conf-enabled/zabbix.conf
修改标准时区为 Asia/Shanghai
```

![](Typoraimage-20220211220534064.png)

### 1.9. 中文显示

```sh
sudo apt install language-pack-zh-hans  #安装中文语言包
sudo vim /etc/locale.gen  #找到zh_CN.UTF-8 UTF-8 并取消#号注释，然后保存并退出
sudo locale-gen  #编译语言包
sudo vim /etc/default/locale #修改默认语言为中文，将原来的内容改为 LANG=zh_CN.UTF-8
```

![](Typoraimage-20220211221904380.png)

### 1.10. 安装出现的问题

#### 1.10.1. Minimum required size of PHP post is 16M (configuration option "post_max_size").

![](image-20220601181428220.png)

**解决步骤：**

```sh
sudo vi /etc/php/8.1/apache2/php.ini
```

**post_max_size**8M `16M`

**max_execution_time**30 `300`

**max_input_time**60 `300`

**date.timezone = Asia/Shanghai**

```sh
sudo systemctl restart zabbix-server zabbix-agent apache2 grafana-server
```

#### 1.10.2. ERROR 1396 (HY000): Operation CREATE USER failed for `'zabbix'@'%'`

```sh
mysql> create user zabbix@`%` identified by '123456';
ERROR 1396 (HY000): Operation CREATE USER failed for 'zabbix'@'%'
```

![](image-20220601194844884.png)

**原因分析**：

1. 已经存在了`zabbix`用户
2. 在执行删除`zabbix`用户的时候没有删除干净

**解决方法：**

重新进行删除。

```bash
drop user zabbix@'%';
flush privileges;
```

### 1.11. 卸载 `zabbix`

 1. 删除软件

    ```sh
    sudo apt-get --purge remove zabbix-server-mysql -y 
    sudo apt-get autoremove zabbix-server-mysql -y 
    
    sudo apt-get --purge remove zabbix-frontend-php -y 
    sudo apt-get autoremove zabbix-frontend-php -y 
    
    sudo apt-get --purge remove abbix-apache-conf -y 
    sudo apt-get autoremove abbix-apache-conf -y 
    
    sudo apt-get --purge remove zabbix-agent -y    #删除软件其配置
    sudo apt-get autoremove zabbix-agent    -y      #删除软件依赖包
    ```

 2. 清理数据

    ```sh
    sudo dpkg -l |grep ^rc|awk '{print $2}' |sudo xargs dpkg -P
    ```

 3. 删除以上apt-get下载的软件包

    ```sh
    sudo apt-get autoclean
    ```

 4. 删除缓存的所有软件包

    ```sh
    sudo apt-get clean
    ```

 5.  删除其他软件依赖的但现在已不用的软件包（保留配置文件）

    ```sh
    sudo apt-get autoremove
    ```

 6. 查询出冗余文件并删除

    ```sh
    sudo find / -name zabbix
    ```

    ![](Typoraimage-20220216161855867.png)

7. 执行rm-rf 删除冗余文件

    ```sh
    sudo rm -rf /run/zabbix
    sudo rm -rf /etc/zabbix
    sudo rm -rf /usr/share/zabbix
    sudo rm -rf /var/log/zabbix
    sudo rm -rf /var/lib/mysql/zabbix
    ```
    
 8.  删除包含`zabbix`关键字的文件或者文件夹

    ```sh
    sudo find / -name "zabbix*" | sudo xargs rm -rf
    ```


##  grafana

### 1.12. 下载grafana deb安装包

```sh
sudo apt-get install -y adduser libfontconfig1
sudo wget https://dl.grafana.com/enterprise/release/grafana-enterprise_8.5.4_amd64.deb
sudo dpkg -i grafana-enterprise_8.5.4_amd64.deb
```

### 1.13. 启动grafana-server

```sh
sudo systemctl restart grafana-server
sudo systemctl enable grafana-server
```

###  安装zabbix插件

```sh
grafana-cli plugins list-remote
sudo grafana-cli plugins install alexanderzobnin-zabbix-app

#重启grafana-server
sudo systemctl restart grafana-server
```

也可以在`grafana->plugins`这里安装

![](image-20220601201553650.png)

![](image-20220601201145104.png)

### 1.14. 登录grafana服务器[10.40.38.67 换成你的ip地址] [用谷歌浏览器或者microsoft Edge浏览器打开]

```sh
http:/10.40.38.67:3000/
#默认用户名和密码为admin、admin
```

### 1.15. grafana 配置zabbix数据源

![](Typora1.png)

### 1.16. grafana 配置zabbix监控面板

![](Typoraimage-20220211222333409.png)

<font color='red'>**在点击完new dashboard 按钮以后 按ctrl + s 保存一个自己定义的仪表盘**</font>

![](Typoraimage-20220211222517727.png)

### 1.17. `grafana`增加主题

```php
安装插件：grafana-cli plugins install yesoreyeram-boomtheme-panel
grafana主题地址：https://github.com/charles1503/grafana-theme/tree/master/CSS/themes/grafanas
grafana更改主题教程：https://www.bilibili.com/read/cv7004400
视频教程：https://cloud.tencent.com/developer/video/11330
http://10.40.38.67:3000/public/themes/aquamarine.css
```

具体操作步骤：

1. 创建一个目录，用于存放下载对应主题的`css`文件

    ```sh
    sudo mkdir /usr/share/grafana/public/themes/
    cd /usr/share/grafana/public/themes/
    ```

    使用一个for 循环下载对应的所有主题`css`文件

    ```sh
    for f in grafana-base.css aquamarine.css hotline.css dark.css plex.css space-gray.css organizr-dashboard.css;do wget https://raw.githubusercontent.com/505384662/grafana-theme/master/CSS/themes/grafana/$f;done
    ```

2. 为`Grafana`安装社区插件`Boom Theme`

    ```sh
    sudo grafana-cli plugins install yesoreyeram-boomtheme-panel
    sudo systemctl restart grafana-server
    ```

3. 在`Dashboard`中添加`Boom Theme`

    ![](e863c9e3bcd7a4801f8aa1e9ad37d8697353b603.png@942w_278h_progressive.webp)

    ![](image-20220601212433764.png)

### 1.18. grafana 主题修改地址

```sh
cd /usr/share/grafana/public/themes
```

![](Typoraimage-20220217192410413.png)

### 1.19. grafana 加时钟

```sh
grafana-cli plugins install grafana-clock-panel
systemctl restart grafana-server
```

### 1.20. grafana flowcharting安装

```sh
sudo grafana-cli plugins install agenty-flowcharting-panel
sudo systemctl restart grafana-server
```

###  grafana 修改模板地址

```php
https://grafana.com/grafana/dashboards
```

![](Typoraimage-20220211232919468.png)

```php
zabbix 修改配置地址：http://192.168.70.130/zabbix/setup.php
zabbix 展示地址：http://192.168.70.130/zabbix/zabbix.php?action=dashboard.view
grafana 展示地址: http://192.168.70.130:3000/d/tYxzFya7z/test_zabbix?orgId=1
```

### 1.21. `Grafana` 匿名访问（免登录）

1. 修改`Grafana`配置文件

    在`Grafana`的配置文件 `/etc/grafana/grafana.ini `中，找到 `[auth.anonymous]` 配置块，将其下的匿名访问控制 `enabled` 设置为 `true`，组织权限设置为 `Viewer`

    `Viewer`:**`只读`**模式

    `Editor`:**`可编辑`**模式

    `Admin`:**`管理员`**模式

    ```sh
    #################################### Anonymous Auth ######################
    # Set to true to disable (hide) the login form, useful if you use OAuth, defaults to false
    disable_login_form = true 
    
    [auth.anonymous]
    # enable anonymous access
    enabled = true
    
    # specify organization name that should be used for unauthenticated users
    org_name = Main Org.
    
    # specify role for unauthenticated users
    org_role = Viewer 
    ```

2. 重启`Grafana`服务

    修改完配置文件，重启`Grafana`服务，命令如下：

    ```sh
    sudo systemctl restart grafana-server
    ```

### 1.22. 卸载 `grafana`

 1. 查找到安装软件名

    ```sh
    sudo dpkg -l | grep grafana
    ```

    ![](Typoraimage-20220216154734448.png)

 2.  删除软件

    ```sh
    sudo dpkg -r grafana-enterprise
    ```

 3. 查询出冗余文件并删除

    ```sh
    find / -name grafana
    ```

    ![Typoraimage-20220216162443274](Typoraimage-20220216162443274.png)

    用rm-rf 命令删除

    ```sh
    rm -rf /etc/grafana
    rm -rf /usr/share/grafana
    rm -rf /usr/share/grafana/public/themes/grafana-theme/CSS/themes/grafana
    rm -rf /var/log/grafana
    rm -rf /var/lib/grafana
    ```

## 2. apache2

### 2.1. apache2启动报错

![Typoraimage-20220213113645141](Typoraimage-20220213113645141.png)

大致意思没有导入`apache` 环境变量 解决办法:

```sh
source /etc/apache2/envvars
```

还是报错

![Typoraimage-20220213113849401](Typoraimage-20220213113849401.png)

大致意思是`80`端口被占用了 我选择的方法是kill占用进程在重启

```sh
root@hls:/root# netstat -lnp|grep 80
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      950/nginx: master p
tcp6       0      0 :::80                   :::*                    LISTEN      950/nginx: master p
unix  2      [ ACC ]     STREAM     LISTENING     41930    1228/zabbix-plugin_  /tmp/plugin835680808
```

```sh
root@hls:/root#  kill -9 950
root@hls:/root# systemctl restart zabbix-server zabbix-agent apache2
```

### 2.2. 卸载apache2

1. 删除软件

   ```
   //1. 删除apache
   sudo apt-get --purge remove apache2
   sudo apt-get --purge remove apache2.2-common
    
   //2.找到没有删除掉的配置文件，一并删除
   sudo find /etc -name "*apache*" |xargs  rm -rf 
   sudo rm -rf /var/www
   sudo rm -rf /etc/libapache2-mod-jk
    
   //3.删除关联，这样就可以再次用apt-get install apache2 重装了
   #dpkg -l |grep apache2|awk '{print $2}'|xargs dpkg -P//注意：这一步可能会报错，但也没关系
   ```

2. 查询出冗余文件并删除

   ```sh
   sudo find / -name apache2
   ```

    ![Typoraimage-20220216162739660](Typoraimage-20220216162739660.png)

3. 用rm -rf 命令删除

   ![Typoraimage-20220216162825900](Typoraimage-20220216162825900.png)

## 3. Nginx

### 3.1. 官网下载地址

```sh
http://nginx.org/en/download.html
```

### 3.2. 一些环境准备

1. 安装编译工具

   ```sh
   sudo apt-get install build-essential 安装编译工具 安装gcc什么的好便于下面编译安装
   ```
   
2. 安装pcre包

   ```sh
   sudo apt-get update
   sudo apt-get install libpcre3 libpcre3-dev
   sudo apt-get install openssl libssl-dev
   ```

3. 安装 zlib 库

   ```sh
   sudo apt install zlib1g-dev
   ```

### 3.3. 下载安装Nginx

```sh
sudo wget http://nginx.org/download/nginx-1.21.6.tar.gz
sudo tar -xzvf nginx-1.21.6.tar.gz
cd nginx-1.21.6
sudo ./configure --prefix=/usr/local/nginx --with-http_ssl_module --with-stream --with-mail=dynamic #最好用 --prefix指定路径，便于后面删除[只需要删除prefix指定的文件夹就行了]，不指定的话后面删除比较麻烦
sudo make 
sudo make install
```

### 3.4. 制作软连接

```sh
ln -s /usr/local/nginx/sbin/nginx /usr/local/sbin/
```

### 3.5. 配置环境变量 编辑`/etc/profile`并且追加Nginx的环境变量

```sh
export NGINX_HOME=/usr/local/nginx
export PATH=$PATH:$NGINX_HOME/sbin
```

![Typoraimage-20220218152713494](Typoraimage-20220218152713494.png)

#### 3.5.1. 生效环境变量

```sh
source /etc/profile
```

### 3.6. 测试是否安装成功

```sh
nginx -v
```

![Typoraimage-20220218152739564](Typoraimage-20220218152739564.png)

### 3.7. 启动Nginx

```sh
sudo nginx
```

### 3.8. 强制停止Nginx

```sh
sudo pkill -9 nginx
```

### 3.9. 查看Nginx进程

```sh
ps aux|grep nginx
```

### 3.10. 配置防火墙

```sh
sudo ufw allow 'Nginx Full'
```

### 3.11. 验证防火墙是否允许 出现下面两种情况都认为可以

```sh
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere
Nginx Full                 ALLOW       Anywhere
22/tcp (v6)                ALLOW       Anywhere (v6)
Nginx Full (v6)            ALLOW       Anywhere (v6)
```

```sh
sudo ufw status
状态：不活动
```

### 3.12. 测试访问

```php
http://192.168.70.132:7000
```

![Typoraimage-20220218131129493](Typoraimage-20220218131129493.png)

### 3.13. Nginx 相关文件位置

```sh
nginx path prefix: "/usr/local/nginx"
nginx binary file: "/usr/local/nginx/sbin/nginx"
nginx modules path: "/usr/local/nginx/modules"
nginx configuration prefix: "/usr/local/nginx/conf"
nginx configuration file: "/usr/local/nginx/conf/nginx.conf"
nginx pid file: "/usr/local/nginx/logs/nginx.pid"
nginx error log file: "/usr/local/nginx/logs/error.log"
nginx http access log file: "/usr/local/nginx/logs/access.log"
nginx http client request body temporary files: "client_body_temp"
nginx http proxy temporary files: "proxy_temp"
nginx http fastcgi temporary files: "fastcgi_temp"
nginx http uwsgi temporary files: "uwsgi_temp"
nginx http scgi temporary files: "scgi_temp"

```

###  卸载 Nginx

```sh
sudo rm -rf /usr/local/nginx
sudo rm -rf /usr/local/nginx/sbin/nginx #软连接也记得删除
如果想完全干净，/etc/profile 配置文件中指定的环境变量也可以删除
```

## 4. mysql

### 4.1. 安装mysql

```shell
sudo apt update
sudo apt install mysql-server
```

安装完成后，MySQL服务将自动启动。要验证MySQL服务器正在运行，请输入：

```shell
sudo systemctl status mysql
```

###  彻底卸载mysql方法

1. 查看依赖包

   ```sh
   dpkg --list | grep mysql
   ```

2. 先依次执行以下命令

   ```sh
   sudo apt-get remove mysql-common
   
   sudo apt-get autoremove --purge mysql-server-5.0    # 卸载 MySQL 5.x 使用,  非5.x版本可跳过该步骤
   
   sudo apt-get autoremove --purge mysql-server
   ```

3. 然后再用 

   ```sh
   dpkg --list | grep mysql 
   ```

4. 查看一下依赖包最后用下面命令清除残留数据

   ```sh
   dpkg -l |grep ^rc|awk '{print $2}' |sudo xargs dpkg -P
   ```

5. 查看从MySQL APT安装的软件列表, 执行后没有显示列表, 证明MySQL服务已完全卸载

   ```sh
   dpkg -l | grep mysql | grep i
   ```

6. 博客地址

   ```php
   https://blog.csdn.net/PY0312/article/details/89481421
   ```

###  MySQL在Ubuntu上启动出错Could not open ‘abstractions/mysql‘

```
rm -rf /etc/apparmor.d/abstractions/mysql 
rm -rf /etc/apparmor.d/cache/usr.sbin.mysqld 
find / -name 'mysql*' -exec rm -rf {} \;
```

### 4.2. 连接MySql报错“can't connect to local mysql server through socket '/var/run/mysqld/mysqld.sock'

```sh
cd /etc/init.d
sudo service mysql stop
sudo service mysql start 
```

###  mysql Ubuntu 20.04   Access denied for user 'root'@'localhost

1. 首先输入以下指令 获取密码：

   ```sh
   sudo cat /etc/mysql/debian.cnf
   ```

   ![Typoraimage-20220214155922420](Typoraimage-20220214155922420.png)

2. 再输入以下指令进入mysql

   ![TyporaTypora20190705101218937](TyporaTypora20190705101218937.png)

2. 查询user关键字段

      ```sql
      select user, authentication_string,plugin,Host from mysql.user;
      ```
   
      ![Typoraimage-20220214172132695](Typoraimage-20220214172132695.png)
   
3. 修改密码格式
   
      ```sql
      use mysql;
      update user set plugin='mysql_native_password' where user='root';
      flush privileges;
      ```
   
4. 修改密码
   
      ```sql
      use mysql;
      ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '123456';
      flush privileges;
      ```
   
5. 输入
   
      ```sh
      mysql -uroot -p123456;
      ```
   
       查看效果

      ![Typoraimage-20220214172442812](Typoraimage-20220214172442812.png)

   

6. 让别的ip能连上wsl数据库

      ```sql
      use mysql;
      update user set Host='%' where user='root';
      flush privileges;
   ```
   
   输入
   
      ```sql
      select user, authentication_string,plugin,Host from mysql.user;
      ```
   
   查看效果
   
      ![Typoraimage-20220214173438761](Typoraimage-20220214173438761.png)
   
7. 开启远程访问

      ```sh
      sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf
       # 注释 bind-address = 127.0.0.1
      ```
   
      ![Typoraimage-20220214173941935](Typoraimage-20220214173941935.png)

8. 重启mysql

      ```sh
      sudo service mysql restart
      ```
   
10. 效果

      ![Typoraimage-20220214174241984](Typoraimage-20220214174241984.png)

## 5. ELK 

### 5.1. 一些准备

#### 5.1.1. 官网地址

```php
 https://www.elastic.co/guide/en/elasticsearch/reference/8.0/deb.html#deb-repo
```

#### 5.1.2. 虚拟机

1. 想要多开最好是克隆一份出来 比如2就是克隆的1的镜像

   ![Typoraimage-20220216150357627](Typoraimage-20220216150357627.png)

2. 修改 克隆的虚拟机网卡地址 

   ```sh
   sudo vim /etc/netplan/00-installer-config.yaml
   ```

   修改内容:

   ```sh
   network:
     ethernets:
       ens33:     #配置的网卡的名称
         addresses: [192.168.70.130/24]    #配置的静态ip地址和掩码
         dhcp4: no    #关闭DHCP，如果需要打开DHCP则写yes
         optional: true
         gateway4: 192.168.70.2 #网关地址
         nameservers:
            addresses: [192.168.70.2,114.114.114.114]    #DNS服务器地址，多个DNS服务器地址需要用英文逗号分隔开
     version: 2
     renderer: networkd    #指定后端采用systemd-networkd或者Network Manager，可不填写则默认使用systemd-workd
   ```
   
3. 使配置生效

   ```sh
   sudo netplan apply
   ```

4. 注意事项

   ```sh
   1、ip地址和DNS服务器地址需要用[]括起来，但是网关地址不需要
   2、注意每个冒号后边都要先加一个空格
   3、注意每一层前边的缩进，至少比上一层多两个空格
   ```

#### 5.1.3. 安装java环境

1.  安装java

   ```sh
   sudo apt install openjdk-8-jdk
   ```

2. 查看java 版本

   ```sh
   sudo java -version
   ```


   3. 查看 java 路径

      ```sh
      sudo which java
      ```

      ![Typoraimage-20220215221138590](Typoraimage-20220215221138590.png)


   4. ls -l /usr/bin/java 看看这是否是个软连接，找出这个软连接指向的路径

      ```
      ls -l /usr/bin/java
      ```

      ![Typoraimage-20220215221258532](Typoraimage-20220215221258532.png)

      的确为软连接，继续往下找指向的路径

      ![Typoraimage-20220216163646385](Typoraimage-20220216163646385.png)

       至此，java 的安装路径即为  **<font color='red'>/usr/lib/jvm/java-11-openjdk-amd64/bin/java</font>**


   5. 配置 java 环境

      ```sh
      sudo vim /etc/profile
      ```


   6. 在弹出的 vim 编辑器中输入

      ```sh
      # JAVA 
      JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
      PATH=$JAVA_HOME/bin:$PATH
      export JAVA_HOME PATH
      ```


   7. esc 退出编辑模式，输入 :x后，单击回车退出。

      在终端输入

      ```sh
      source /etc/profile
      ```

       使之前的配置生效。


   8. 验证

      java -version

      ![Typoraimage-20220216164233792](Typoraimage-20220216164233792.png)

      $JAVA_HOME/bin/java -version

      ![Typoraimage-20220216164211060](Typoraimage-20220216164211060.png)

#### 5.1.4. python3

  **<font color='red'>[不是必须装主要是想使用 json.tool 格式化输出]</font>**

 1. 安装python3.8

    ```sh
    sudo apt-get install python3.8
    ```

2.  建立软连接

   ```sh
   sudo ln -s /usr/bin/python3.8 /usr/bin/python
   ```

3. 如果想要删除软连接

   ```sh
   sudo rm -rf /usr/bin/python
   ```
   
4. 格式化输出

   ```php
   curl -XGET http://192.168.70.131:9200/_mapping | python -m json.tool
   ```

### 5.2. Elasticsearch

#### 5.2.1. 基础知识

#####  和关系型数据库的比较

|   DBMS   |           Elasticsearch            |
| :------: | :--------------------------------: |
| database |               Index                |
|  table   |  type(在7.0之后type为固定值_doc)   |
|   Row    |              Document              |
|  Column  |               Field                |
|  Schema  |              Mapping               |
|   SQL    | DSL(Descriptor Structure Language) |

####  安装Elasticsearch

1. deb包安装方式

   ```sh
   sudo wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.2.2-amd64.deb
   sudo wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.2.2-amd64.deb.sha512
   shasum -a 512 -c elasticsearch-8.2.2-amd64.deb.sha512 
   sudo dpkg -i elasticsearch-8.2.2-amd64.deb
   ```

2. 执行**<font color='red'>sudo dpkg -i elasticsearch-8.2.2-amd64.deb</font>** 回生成超级用户密码 **<font color='red'>0NgzdrlHquc1YdXrQout</font>**

   ```sh
   --------------------------- Security autoconfiguration information ------------------------------
   
   Authentication and authorization are enabled.
   TLS for the transport and HTTP layers is enabled and configured.
   
   The generated password for the elastic built-in superuser is : 0NgzdrlHquc1YdXrQout
   
   If this node should join an existing cluster, you can reconfigure this with
   '/usr/share/elasticsearch/bin/elasticsearch-reconfigure-node --enrollment-token <token-here>'
   after creating an enrollment token on your existing cluster.
   
   You can complete the following actions at any time:
   
   Reset the password of the elastic built-in superuser with
   '/usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic'.
   
   Generate an enrollment token for Kibana instances with
    '/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana'.
   
   Generate an enrollment token for Elasticsearch nodes with
   '/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s node'.
   
   -------------------------------------------------------------------------------------------------
   ```
   
3. 生成 ca 、生成 证书

   ```sh
   # 生成 ca
   # 根据提示：
   # 输入 ca 的密码（密码不要忘记，后面生成证书需要）
   # 输入生成 ca 的文件名（默认会让你输入 elastic-stack-ca.p12，这里就按照默认的来）
   sudo /usr/share/elasticsearch/bin/elasticsearch-certutil ca
   
   # 生成证书
   # 根据提示：
   # 输入之前 ca 的密码
   # 输入生成证书的文件名（默认让你输入 elastic-certificates.p12，这里就按照默认的来）
   # 输入生成证书的密码（密码不要忘记，这个密码在配置 ES keystore 的时候需要）
   # --ca 后面的文件是上面步骤生成的 elastic-stack-ca.p12 文件，如果修改了的话，这里也需要修改
   sudo /usr/share/elasticsearch/bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12
   ```
   
   ![](Typoraimage-20220216111441217.png)
   
   ![Typoraimage-20220216111555291](Typoraimage-20220216111555291.png)

    ![](image-20220602100009735.png)

​		为了方便管理，一般将 *ca* 与证书放到 *`~/.config/certs`* 目录下

- ```sh
  # 创建目录并移动 ca 与证书
  sudo mkdir -p ~/.config/certs && sudo mv /usr/share/elasticsearch/elastic-stack-ca.p12 /usr/share/elasticsearch/elastic-certificates.p12 ~/.config/certs
  ```

#### 5.2.2. 启动 Elasticsearch

​	**<font color='red'>[为了安全考虑Elasticsearch不允许使用root用户来启动]</font>**

1. 打开 elasticsearch 配置文件 

   ```sh
   sudo vim /etc/elasticsearch/elasticsearch.yml #打开配置文件
   ```

2. 修改 **<font color='red'>netWork.host</font>**, **<font color='red'>http.port</font>** 字段

   ```yml
   network.host: 10.40.38.66 #注意 network.host:和10.40.38.66 之间需要空格要不启动会有问题，因为配置文件类型为key-vale格式
   ```
   
   ```yml
   http.port: 9200 #注意 http.port:和9200 之间需要空格要不启动会有问题，因为配置文件类型为key-vale格式
   ```
   
   ![](Typoraimage-20220215182019551.png)
   
2. 因为是内网测试暂时关闭 xpack 安全验证方面选项,以后需要再去开启

   ![](image-20220602101815651.png)
   
4. 启动Elasticsearch

   ```sh
   sudo systemctl start elasticsearch.service
   ```
   
5. 开机启动elasticsearch

   ```sh
   sudo systemctl enable elasticsearch.service
   ```

#### 5.2.3. 连接grafana

![](Typoraimage-20220217213204621.png)

#### 5.2.4. Elasticsearch 操作命令 

1. 用jps命令关闭Elasticsearch

   ```sh
   $ jps | grep Elasticsearch
   14542 Elasticsearch
   kill -9 14542
   ```

2. 查看 Elasticsearch 端口

   ```sh
   sudo netstat -tnlp |grep java
   ```

   ![](Typoraimage-20220215213238459.png)

3. 检测是否启动成功

   ```sh
   curl -XGET 'http://192.168.70.131:9200/' -H 'Content-Type: application/json'
   ```

6. 用journal 查看系统日志

   ```sh
   sudo journalctl -f
   ```

   ![](Typoraimage-20220215131826206.png)

7. 用 journal 查看elasticsearch 服务日志

   ```sh
   sudo journalctl --unit elasticsearch
   ```

   ![](Typoraimage-20220215132103390.png)

8. 用journal 查看elasticsearch 指定时间范围的日志

   ```sh
   sudo journalctl --unit elasticsearch --since  "2022-02-01 18:17:16"
   ```

   ![](Typoraimage-20220215132318818.png)
   
7. 查看 elasticsearch.log 

   ```sh
   sudo vim /var/log/elasticsearch/elasticsearch.log
   ```

#### 5.2.5. Elasticsearch 卸载

```sh
 # 查看安装的软件
 sudo dpkg -l | grep elasticsearch 
 #查看安装关联
 sudo dpkg -L  elasticsearch
 #移除安装软件
 sudo dpkg -P elasticsearch 
 #继续查看未卸载的目录和文件
 sudo find / -name elasticsearch
 #移除目录和文件具体参考自己的环境
 sudo rm -rf /var/lib/elasticsearch &&
 sudo rm -rf /var/lib/dpkg/info/elasticsearch.* &&
 sudo rm -rf /etc/default/elasticsearch &&
 sudo rm -rf /etc/init.d/elasticsearch &&
 sudo rm -rf /var/log/elasticsearch &&
 sudo rm -rf /usr/share/elasticsearch
 #在此查看是否有关联的目录和文件
sudo find / -name elasticsearch
```

### 5.3. Logstash

#### 5.3.1. 安装 Logstash

1. 下载安装公共签名

   ```sh
   sudo wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
   ```

2. 接下安装 `apt-transport-https` 包

   ```sh
   sudo apt-get install apt-transport-https
   ```

3. 将存储库保存到 `/etc/apt/sources.list.d/elastic-8.x.list`

   ```sh
   echo "deb https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-8.x.list
   ```

4. 然后你就能安装Elasticsearch了

   ```sh
   sudo apt-get update && sudo apt-get install logstash
   ```

#### 5.3.2. 插件地址

```php
https://www.elastic.co/guide/en/logstash-versioned-plugins/current/index.html
```

#### 5.3.3. 配置表字段解释

```php
https://blog.csdn.net/weixin_42073629/article/details/110154037?utm_medium=distribute.pc_aggpage_search_result.none-task-blog-2~aggregatepage~first_rank_ecpm_v1~rank_v31_ecpm-1-110154037.pc_agg_new_rank&utm_term=logstash%E5%8F%82%E6%95%B0convert&spm=1000.2123.3001.4430
```

#### 5.3.4. 查看安装的插件

```sh
sudo /usr/share/logstash/bin/logstash-plugin list
```

####  启动Lostash	

   1. 修改 logstash.yml 配置

      ```sh
      sudo vim /etc/logstash/logstash.yml
      ```
      
      ![](Typoraimage-20220218001656526.png)

#### 5.3.5. 导入数据[利用logstash 直接分析movies.csv 传送给elasticsearch方式]

​		  收集流程: **<font color='red'>movies.csv->logstash->elasticdearch->grafana</font>**

 1. 下载ml-latest.zip 数据

    ```sh
    sudo wget https://files.grouplens.org/datasets/movielens/ml-latest.zip
    ```

 2. 解压 ml-latest.zip

    ```sh
    sudo unzip ml-latest.zip
    ```

 3. 在/etc/logstash 目录下创建logstash.conf 文件

    ```sh
    sudo vim /etc/logstash/logstash.conf
    ```

 4. 把以下内容写入logstash.conf

    ```sh
    input {
      file {
      	 #监听文件的路径
        path => "/home/hls/downs/ml-latest/movies.csv"
        #监听文件的起始位置，默认是end
        start_position => "beginning"
         #监听文件读取信息记录的位置
        sincedb_path => "/home/hls/downs/ml-latest/db_path.log"
      }
    }
    filter {
      csv {
        separator => ","
        columns => ["id","content","genre","@timestamp"]
      }
    
      mutate {
       # split => { "genre" => "|" }
       # remove_field => ["path", "host","@timestamp","message"] #删除无用字段
      }
    
      mutate {
        split => ["content", "("] #左括号分割
        add_field => { "title" => "%{[content][0]}"} #增加字段
        add_field => { "year" => "%{[content][1]}"} #增加字段
      }
    
      mutate {
        convert => { #year 转换成整型
          "year" => "integer"
        }
        strip => ["title"] #去掉字段首尾的空格
       # remove_field => ["path", "host","@timestamp","message","content"]  #删除无用字段
      }
    }
    output {
       elasticsearch {
       	 # 双引号中的内容为ES的地址，视实际情况而定
         hosts => "http://192.168.70.131:9200"
         index => "movies"
         document_id => "%{id}" #docId 等价于_id 字段
       }
      stdout {}
    }
    ```
    
 5. 如果需要重新导入，先删除db_path.log 文件

    ```sh
    sudo rm -rf /var/lib/logstash/.lock
    sudo rm -rf /home/hls/downs/ml-latest/db_path.log
    sudo /usr/share/logstash/bin/logstash -f /etc/logstash/logstash.conf
    ```
    
 6. 报错

    1. 执行命令**<font color='red'>sudo /usr/share/logstash/bin/logstash -f /etc/logstash/logstash.conf</font>** 后如果报错
    
       ```sh
       WARNING: Could not find logstash.yml which is typically located in $LS_HOME/config or /etc/logstash. You can specify the path using --path.settings. Continuing using the defaults
       ```

       那么就创建软连接
    
       ```sh
       cd /usr/share/logstash
       sudo ln -s /etc/logstash ./config
       ```
    
    2. 执行命令**<font color='red'>sudo /usr/share/logstash/bin/logstash -f /etc/logstash/logstash.conf</font>** 后如果报错
    
       ```sh
        Logstash could not be started because there is already another instance using the configured data directory.  If you wish to run multiple instances, you must change the "path.data" setting.
       ```
    
        那么就去  logstash.yml 中path.data 指定的路径上去删除.lock文件
    
       ```sh
       cd /var/lib/logstash
       sudo ls -a
       sudo rm -rf .lock
       ```
    
       或者直接一句话
    
       ```sh
       sudo rm -rf /var/lib/logstash/.lock
       ```
    
       ![](Typoraimage-20220217124057908.png)
    
       ![](Typoraimage-20220217124136069.png)
    

#### 5.3.6. 强制查看输出  **<font color='red'>logstash.conf</font>** 修改成你自己的文件

```sh
sudo /usr/share/logstash/bin/logstash  /etc/logstash/logstash.conf --verbose --debug
```

#### 5.3.7. 查看数据

 1. 用Kibana的命令行工具执行 **<font color='red'>GET _cat/indices</font>** 命令，就能看见导入到Elasticsearch的索引

    ![](Typoraimage-20220217112819405.png)
    
 2. 用kibana的命令行工具执行**<font color='red'>GET /lua_cpu_monitor-2022.06.03/_search</font>**命令,就能看见导入到Elasticsearch的数据
    
       ![](image-20220603204457448.png)

#### 5.3.8. 自动重新加载配置命令

**<font color='red'>logstash.conf</font>** 修改成你自己的文件

```sh
sudo /usr/share/logstash/bin/logstash  /etc/logstash/logstash.conf --config.reload.automatic
```

默认检测时间是**<font color='red'>3</font>**秒 可以通过下列命令修改 把<>号里面的2换成你想要的时间

```sh
sudo /usr/share/logstash/bin/logstash  /etc/logstash/logstash.conf --config.reload.interval <2>
```

#### 5.3.9. 卸载Logstash

```sh
 # 查看安装的软件
 sudo dpkg -l | grep logstash
 #查看安装关联
 sudo dpkg -L  logstash
 #移除安装软件
 sudo dpkg -P logstash 
 #继续查看未卸载的目录和文件
 sudo find / -name logstash
 #移除目录和文件具体参考自己的环境
 sudo rm -rf /var/lib/logstash &&
 sudo rm -rf /var/lib/dpkg/info/logstash.* &&
 sudo rm -rf /etc/default/logstash &&
 sudo rm -rf /etc/init.d/logstash &&
 sudo rm -rf /etc/logstash &&
 sudo rm -rf /var/log/logstash &&
 sudo rm -rf /usr/share/logstash
 #在此查看是否有关联的目录和文件
sudo find / -name logstash
```

### 5.4. Kibana

#### 5.4.1. 安装Kibana

 1. 下载安装公共签名

    ```sh
    wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
    ```

2. 接下安装 apt-transport-https 包

   ```sh
   sudo apt-get install apt-transport-https
   ```

3. 将存储库保存到 `/etc/apt/sources.list.d/elastic-8.x.list`

   ```sh
   echo "deb https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-8.x.list
   ```
   
4. 然后你就能安装Kibana了

   ```sh
   sudo apt-get update && sudo apt-get install kibana
   ```

#### 5.4.2. 启动Kibana

 1. 打开kibana.yml 文档

    ```sh
    sudo vim /etc/kibana/kibana.yml
    ```

    修改 **<font color='red'>server.port,server.host</font>** 字段

    ![](Typoraimage-20220216215646322.png)

    

 2. 启动

    ```sh
    sudo systemctl start kibana.service
    ```

 3. 自启动

    ```sh
    sudo systemctl enable kibana.service
    ```

 3. 查看 kibana日志

    ```sh
    sudo vim /var/log/kibana
    ```

 4. 用谷歌或者微软自带浏览器打开地址

    ```php
    http://10.40.38.66:5601
    ```


#### 5.4.3. 卸载Kibana

```sh
 # 查看安装的软件
 sudo dpkg -l | grep kibana
 #查看安装关联
 sudo dpkg -L kibana
 #移除安装软件
 sudo dpkg -P kibana 
 #继续查看未卸载的目录和文件
 sudo find / -name kibana
 #移除目录和文件具体参考自己的环境
 sudo rm -rf /var/lib/kibana &&
 sudo rm -rf /var/lib/dpkg/info/kibana.* &&
 sudo rm -rf /etc/kibana
 #在此查看是否有关联的目录和文件
sudo find / -name kibana
```

### 5.5. Filebeat

**搭配filebeat主要使用收集nginx数据, `和上面的利用logstash解析movies.csv，然后收集数据给elasticsearch`的方式不一样** 

收集流程: **<font color='red'>nginx->filebeat->logstash->elasticdearch->grafana</font>**

#### 5.5.1. 安装Filebeat

```sh
sudo curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.2.2-amd64.deb
sudo dpkg -i filebeat-8.2.2-amd64.deb
```

#### 5.5.2. 修改 filebat.yml 配置文件

```sh
sudo vim /etc/filebeat/filebeat.yml
```

修改下列几项

```yml
#  ============================== Filebeat inputs ===============================
filebeat.inputs:
- type: filestream
  id: my-filestream-id 
  enabled: true 
  paths:
    - /home/hls/work/blueprint-server-runtime/log/lua_cpu_monitor.log 
  tags: ["lua_cpu_monitor_log"]

- type: filestream 
  id: my-filestream-id 
  enabled: true 
  paths:
    - /home/hls/work/blueprint-server-runtime/log/lua_mem_monitor.log 
  tags: ["lua_mem_monitor_log"]

#  ============================== Filebeat modules ==============================
filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false
  
setup.template.settings:
  index.number_of_shards: 1

#  ------------------------------ Logstash Output -------------------------------
output.logstash:
  # The Logstash hosts
  hosts: ["10.40.38.66:5555"]

#  ================================= Processors =================================
processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_docker_metadata: ~
  - add_kubernetes_metadata: ~
```

#### 6.5.3. 测试filebeat启动后，查看相关输出信息

```sh
sudo filebeat -e -c /etc/filebeat/filebeat.yml -d "publish"
```

#### 6.5.4. 后台方式启动filebeat

```sh
nohup filebeat -e -c /etc/filebeat/filebeat.yml >/dev/null 2>&1 & #将所有标准输出及标准错误输出到/dev/null空设备，即没有任何输出
```

```sh
nohup filebeat -e -c /etc/filebeat/filebeat.yml > filebeat.log &
```

#### 6.5.5. 停止filebeat

```sh
ps -ef | grep filebeat
kill -9 进程号
```
#### 6.5.6. 启动出现的问题

执行命令`systemctl start filebeat.service`就能够启动了。而后执行`ps -ef|grep filebeat`查看一下

能够看到已经启动胜利了，如果你发现没有启动成功，那么就执行` cd /usr/bin`，在这个目录下执行`./filebeat -c /etc/filebeat/filebeat.yml -e`，这样会提醒具体的错误信息。而用`systemctl start filebeat.service`启动的时候没有任何提醒，连在 /var/log/filebeat/ 和 /var/lib/filebeat/registry/filebeat/ 都没找到错误信息，这里属实有点坑。

重新启动命令`systemctl restart filebeat.service`

#### 6.5.7 去安装logstash的机器启动logstash

1. 增加 logstash_filebeat.conf 文档

    ```sh
    sudo vim /etc/logstash/conf.d/logstash_filebeat.conf
    ```

    把以下内容粘贴上保存

    ```yml
    input {
            beats {
                    port => 5555 #这个地址不能和logstash.yml 里面的api.http.host: 9600 一样，要不会出现地址已经被绑定的错误
            }
    }
    
    output {
     if "lua_cpu_monitor_log" in [tags] {
        elasticsearch {
            hosts => ["10.40.38.66:9200"]
            index => "lua_cpu_monitor-%{+YYYY.MM.dd}"
        }
      }
      
    if "lua_men_monitor_log" in [tags] {
        elasticsearch {
            hosts => ["10.40.38.66:9200"]
            index => "lua_men_monitor-%{+YYYY.MM.dd}"
        }
      }
    } 
    ```

2. 重新加载新的配置并启动logstash

    **<font color='red'>先启动logstash，然后在启动filebeat，不然的话filebeat会找不到beats插件的:5555端口</font>**

    ```sh
    sudo rm -rf /var/lib/logstash/.lock
    sudo /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/logstash_filebeat.conf --verbose --debug
    ```

#### 6.5.8. 用filebeat 监控 nginx

 1. 修改 nginx conf 配置表

    ```sh
    sudo vim /usr/local/nginx/conf/nginx.conf
    ```

 2. 加入如下日志格式

    ```sh
    log_format  main  '{"@timestamp":"$time_iso8601",'
                      '"@source":"$server_addr",'
                      '"hostname":"$hostname",'
                      '"ip":"$remote_addr",'
                      '"client":"$remote_addr",'
                      '"request_method":"$request_method",'
                      '"scheme":"$scheme",'
                      '"domain":"$server_name",'
                      '"referer":"$http_referer",'
                      '"request":"$request_uri",'
                      '"args":"$args",'
                      '"size":$body_bytes_sent,'
                      '"status": $status,'
                      '"responsetime":$request_time,'
                      '"upstreamtime":"$upstream_response_time",'
                      '"upstreamaddr":"$upstream_addr",'
                      '"http_user_agent":"$http_user_agent",'
                      '"https":"$https"'
                      '}';
    ```

 2. 对比修改下图对应的3个红框地方

    ![](Typoraimage-20220218160818419.png)
    
 4. 重启 nginx

    ```sh
    sudo pkill -9 nginx && sudo nginx
    ```

 4. 用 **<font color='red'>http:192.168.70.132:7000</font>** 登录nginx 网站生成登录日志，然后打开 **<font color='red'>access.log</font>** 日志

    ```sh
    sudo vim /usr/local/nginx/logs/access.log
    sudo tail -f /usr/local/nginx/logs/access.log
    ```
    
    ![](Typoraimage-20220218161633527.png)

#### 6.5.9. 卸载Filebeat

```sh
 # 查看安装的软件
 sudo dpkg -l | grep filebeat
 #查看安装关联
 sudo dpkg -L  filebeat
 #移除安装软件
 sudo dpkg -P filebeat 
 #继续查看未卸载的目录和文件
 sudo find / -name filebeat
 #移除目录和文件具体参考自己的环境
 sudo rm -rf /var/lib/filebeat &&
 sudo rm -rf /var/log/filebeat/filebeat &&
 sudo rm -rf /var/log/filebeat &&
 sudo rm -rf /usr/share/filebeat
 #在此查看是否有关联的目录和文件
sudo find / -name filebeat
```





   

   



