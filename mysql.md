#### centos7 安装mysql-8.0.15-1.el7.x86_64.rpm-bundle.tar(离线安装)

* 解压： tar -xvf mysql-8.0.15-1.el7.x86_64.rpm-bundle.tar
* 文件安装顺序：common --> libs --> client --> server --> devel。(devel可不装)
* 安装命令：rpm -ivh xxx.rpm
* 当提示“mariadb-libs 被 mysql-community-libs-8.0.15-1.el7.x86_64 取代”，是lib和系统自带的冲突，删除后继续：yum remove mysql-libs -y（cnetos7以后，对mysql收费，不支持mysql，而支持mariadb）
* 当提示“net-tools 被 mysql-community-server-8.0.15-1.el7.x86_64 需要”，直接安装缺失的依赖：yum install net-tools -y
* 查找初始密码：
    * 启动： service mysqld start
    * cat /var/log/mysqld.log 
  ```
    * 2020-06-03T01:41:57.036935Z 5 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: e0VCFzdknF;i
  ```

* 初始密码登录：mysql -uroot -p(默认密码登陆后，需修改密码)
* 加密方式（2种）:
    * mysql_native_password（V5.x）
    * caching_sha2_password（V8.x）
* 修改加密方式：vim /etc/my.conf
    * default-authentication-plugin=mysql_native_password(设置对相应的加密方式)
* 修改密码：
```
ALTER USER 'root'@'localhost' IDENTIFIED BY 'Adsl.2020';#注意位数和种类至少大+写+小写+符号+数字
```
* 忘记密码重置：
  vim /etc/my.cnf #注：windows下修改的是my.ini
  skip-grant-tables# 在[mysqld]后面任意一行添加           skip-grant-tables用来跳过密码验证的过程;设置完密码记得删除
  systemctl restart mysqld.service #重启mysql ，就可以免密码登陆了，然后进行修改密码
  

#### yum 安装mysql

* 修改安装
yum-config-manager --disable mysql80-community #关闭8.0版本
yum-config-manager --enable mysql57-community #开启5.7版本
* 将mysql 服务加入开机启动项，并启动mysql进程
    * systemctl enable mysqld.service
    * systemctl start mysqld.service

* 常用mysql服务命令：

    * mysql -u username -p #登录mysql
    * quit #退出mysql 
    * systemctl start mysqld.service  #启动mysql
    * systemctl stop mysqld.service #结束
    * systemctl restart mysqld.service #重启
    * systemctl enable mysqld.service #开机自启
    * select version(); #查看mysql版本
* 开启mysql远程服务
    * 修改mysql数据库下的user表中host的值
  可能是你的帐号不允许从远程登陆，只能在localhost。这个时候只要在localhost的那台电脑，登入mysql后，更改 "mysql" 数据库里的 "user" 表里的 "host" 项，从"localhost"改称"%"登录mysql数据库 执行如下命令：

    ```
        mysql -u root -p
        use mysql;
        update user set host='%' where user='root';
    ```
    * 使用授权的方式
    赋予任何主机访问数据的权限
    ```
    mysql> GRANT ALL PRIVILEGES ON *.* TO root@'%';
    mysql>FLUSH PRIVILEGES
    ``` 
    * 如果想myuser用户使用mypassword密码从任何主机连接到mysql服务器的话
    ```
    GRANT ALL PRIVILEGES ON *.* TO 'myuser'@'%'IDENTIFIED BY 'mypassword' WITH GRANT OPTION;
    ```
    * 如果你想允许用户myuser从ip为192.168.1.6的主机连接到mysql服务器，并使用mypassword作为密码
    ```
    GRANT ALL PRIVILEGES ON *.* TO 'myuser'@'192.168.1.3'IDENTIFIED BY 'mypassword' WITH GRANT OPTION;
    ```
  