title: "Ubuntu上安装postgresql"
date: 2015-06-28 09:58:00
categories: 
- Tech
tags: 
- postgresql
---

[Postgresql](http://www.postgresql.org/)是一个开源的关系型数据库。个人觉得这个东西的网站不是很人性化。doc很全，但是在安装手册里面并不能很快地找到自己想要的内容。于是，在安装了三遍之后，我决定再安装一遍。<!--more-->

## \# 使用apt-get
我的环境是：Ubuntu trusty 64。由于apt默认源上没有9.4版本的postgresql，所以根据[官方安装文档](http://www.postgresql.org/download/linux/ubuntu/)，添加了apt源，使用`sudo apt-get install postgres-9.4`，安装了postgresql系列的软件。运行的命令是：

``` bash
deb http://apt.postgresql.org/pub/repos/apt/ trusty-pgdg main //我的系统是ubuntu trusty
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | \
  sudo apt-key add -
  sudo apt-get update
  sudo apt-get install postgres-9.4
```
官方推荐使用一个专门的linux user来启动，管理postgresql，鉴于实验的目的，所以直接使用当前user了。

### 如何验证安装成功？
查看/usr/lib/postgresql/9.4/bin，首先，目录存在，其次里面有很多命令，包括initdb, pg_ctl, pg_dump, psql等。

使用apt-get安装，会默认启动postgresql server。使用`ps aux | grep postgres`可以看到一条：
`runsv postgresql`

此时，可以尝试使用下文的psql连接server。

## \# 下载deb包进行安装（推荐方式）
在官方下载页[http://www.postgresql.org/download/linux/ubuntu/](http://www.postgresql.org/download/linux/ubuntu/)中的‘Cross distribution packages’部分，可以到[OpenSCG](http://community.openscg.com/se/postgresql/packages.jsp)下载deb包。我使用的是64位Ubuntu，所以下载了9.4.4版本的64位deb。

### 安装
``` text
sudo dpkg -i bigpostgres_9.4.4-1.amd64.openscg.deb
```
安装成功后，会出现提示，根据提示，运行：
``` text
sudo /etc/init.d/bigpostgres-9.4-openscg start
```
之后会问你端口，password，开机启动等问题。我这里选的是端口选择默认端口，password设置成123(一定要设置密码)，开机启动选n。成功之后会有提示。这个时候server实际上已经在运行了，使用`ps`命令看看：
``` text
ps aux | grep postgres
```
可以看到好几个进程。

既然server已经启动了，那么就使用psql连接一下。首先看看当前的ubuntu user是谁。

``` text
whoami
```
我这里输出的是‘vagrant’，由于postgresql建议使用一个叫做postgres的linux user来管理所有关于postgresql的资源，执行postgresql相关命令。所以我们需要切换到postgres这个用户。之前安装的时候，安装包已经给我们创建了这个postgres用户了，所以只需要切换到这个用户就可以了。尝试：
``` text
su postgres
```
提示输入密码，密码不知道是什么。输入：
```
sudo passwd postgres
```
更改postgres用户的密码。之后使用`su postgres`就可以切换到postgres用户了。

postgres被安装到了这个目录：
``` text
/opt/big/postgres/9.4
```
进入这个目录，运行：
``` text
source pg94-openscg.env
```
这个实际上就是设置一些环境变量，PATH什么的。设置好了之后，就可以直接执行：
``` text
psql

```
不加任何参数，表示使用默认配置。这时候提示输入密码，输入上面设置的密码123。这时应该可以成功连接上数据库。

当重启之后，postgresql server就不再运行了，这时候我们可以：
首先`su postgres`
然后`/opt/big/postgres/9.4/bin/postgres -D /opt/big/postgres/9.4/data`，启动server。(使用这种方式的可以略过下一部分“启动server”)

## \# 启动server
在安装成功后，需要建立一个database cluster。所谓database cluster，就是：database cluster: database storage area on disk, a collection of databases managed by a single instance of a running server。在磁盘上建立数据库所需要的空间。创建文件夹`/usr/local/postgres/data`，我将在这个文件夹下建立database cluster。
在/usr/lib/postgresql/9.4/bin下面运行：

``` text
./initdb -D /usr/local/postgres/data/
```
显示错误：

``` text
The files belonging to this database system will be owned by user "vagrant".
This user must also own the server process.

initdb: invalid locale settings; check LANG and LC_* environment variables
```
原因是系统LANG和postgresql需要的不一致，于是：
``` bash
export LANGUAGE="en_US.UTF-8"
export LC_ALL="en_US.UTF-8"
```
再次运行`./initdb -D /usr/local/postgres/data/`。显示：
``` text
The files belonging to this database system will be owned by user "vagrant".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

fixing permissions on existing directory /usr/local/postgres/data ... initdb: could not change permissions of directory "/usr/local/postgres/data": Operation not permitted
```
initdb没有足够的权限访问/usr/local/postgres/data。更改data文件夹的权限：
``` text
sudo chmod 777 /usr/local/postgres/data/
sudo chown vagrant /usr/local/postgres/data //vagrant is my current user, you can get it from the command whoami
```
再次运行`./initdb -D /usr/local/postgres/data/`。显示：
``` text
Success. You can now start the database server using:

    ./postgres -D /usr/local/postgres/data/
or
    ./pg_ctl -D /usr/local/postgres/data/ -l logfile start
```
成功，按照提示，运行：
``` text
./postgres -D /usr/local/postgres/data/
```

ubuntu下面建立一个新的user：
```
adduser postgres
```

使用passwd改变一个user的密码：
```
passwd postgres
```


## \# 使用psql连接到server

输入：
```
psql -U postgres -h localhost
```
如果提示需要密码，则输入postgres这个用户的密码。或者使用之前安装时候的user的账号和密码来进行登录。这都不行的时候，在data文件夹下打开`pg_hba.conf`。把
`local   all             all                                     md5`中的`md5`改为`trust`;**重启** server，然后登录。
登录成功之后，可以更改用户的密码：
```
ALTER USER postgres with password 'YourNewPassword'; // postgres是用户名
```
之后将`pg_hba.conf`改回来，重启server，再次用psql登录时，就可以使用新密码了。

## \# psql的常用指令
在使用以`\`开头的指令的时候不需要最后加上`;`，其他的需要加上分号，表示语句结束。

| requirement              |    command                |
|--------------------------|---------------------------|
| create database          | CREATE DATABASE name      |
| delete databse           | DROP DATABASE [ IF EXISTS ] name |
| show current database    | select current_database();|
| switch database          | \connect database_name    |
| show tables              | \dt                       |
| show current user        | select from current_user; |
| describe table           | \d                        |
| help                     | \h                        |
| quit                     | \q                        |
| show schemas             | \dn                       |
| list all databases       | \l                        |
| list all tables;         | \dt                       |
