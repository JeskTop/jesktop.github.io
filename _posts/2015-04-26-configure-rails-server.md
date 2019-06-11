---
layout: post
title: Rails和PostgreSQL搭建方式
date: 2015-04-26 16:38:40
disqus: y
---

- - -
参考原文：[Passenger/Nginx/Ubuntu/MySQL详尽部署Rails 4.2.3/Ruby2.2.2](http://www.cnblogs.com/jesktop/archive/2012/02/23/2364674.html)

上面的原文是我早之前自己配置环境时所写的文章，一直都有在更新。现在由于现在我从MySQL迁移到PostgreSQL了，而且自己最近在搭建一个服务器，所以把搭建起来的过程记录下来。

{% highlight html %}
服务器提供商：Linode
修改时间：2015-04-26
Ubuntu版本：14.04.2 LTS
Ruby版本：2.6.3
Rails版本：4.2.3
PostgreSQL版本：9.4.1
Nginx版本：1.9.0
{% endhighlight %}

## 更新源和安装依赖包

### 更新源和校正时区
{% highlight html %}
sudo apt-get update
sudo apt-get upgrade
sudo dpkg-reconfigure tzdata   #! 选择Asia，然后再选择自己所在的时区【shanghai】
{% endhighlight %}

### 安装所需的linux包
{% highlight html %}
sudo apt-get install build-essential bison openssl libreadline-dev curl git zlib1g zlib1g-dev libssl-dev libyaml-dev  libxml2-dev libxslt1-dev autoconf libc6-dev zlib1g-dev libssl-dev build-essential curl libc6-dev g++ gcc gnupg2
{% endhighlight %}

### 添加一个rails用户和一个passenger用户组
{% highlight html %}
sudo addgroup server
sudo adduser deploy
sudo usermod -G server,www-data,sudo deploy
su - deploy
{% endhighlight %}

## 安装Ruby和Rails（使用RVM）

### 安装 rvm（可以进行ruby版本控制）
{% highlight html %}
gpg2 --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB
\curl -sSL https://get.rvm.io | bash -s stable
{% endhighlight %}

安装完毕后，重启终端，可以根据以下这个命令看一下是否安装成功：
{% highlight html %}
rvm –v
{% endhighlight %}

### 安装Ruby
{% highlight html %}
rvm install 2.6.3  #! 2.6.3为ruby的版本
{% endhighlight %}

安装完成后，需要设置默认的Ruby版本：
{% highlight html %}
rvm 2.6.3 --default   #! 设置2.2.2为默认的版本
ruby –v               #! 查看当前ruby的版本
{% endhighlight %}

### 安装Rails
{% highlight html %}
$gem install rails   #! 自动安装当期最新版本
{% endhighlight %}

Rails里面自带着一个服务器，方便使用的时候进行测试，开启的命令是rails server。现在我们安装相关的支持：
{% highlight html %}
sudo apt-get install openssl libssl-dev 
sudo apt-get install libopenssl-ruby1.9.1
{% endhighlight %}

## 搭建PostgreSQL

参考：
[How To Install PostgreSQL 9.4 And phpPgAdmin On Ubuntu 14.10](http://www.unixmen.com/install-postgresql-9-4-phppgadmin-ubuntu-14-10/)
[wiki PostgreSQL](https://wiki.postgresql.org/wiki/Apt)

因为ubuntu源所带的PostgreSQL版本，仍然是9.3.x的，所以我们需要更换PostgreSQL的源进行安装。

### 修改获取PostgreSQL的源
添加文件`/etc/apt/sources.list.d/pgdg.list`，并在文件内加上：
{% highlight html %}
deb http://apt.postgresql.org/pub/repos/apt/ trusty-pgdg main
{% endhighlight %}

### 导入repository key
{% highlight html %}
sudo apt-get install wget ca-certificates
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt-get update
sudo apt-get upgrade
{% endhighlight %}

### 安装Postgresql 9.4
{% highlight html %}
sudo apt-get install postgresql-9.4
{% endhighlight %}

### 添加新用户和新数据库
初次安装后，默认生成一个名为postgres的数据库和一个名为postgres的数据库用户。这里需要注意的是，同时还生成了一个名为postgres的Linux系统用户。
因为考虑到安全问题，建议新建一个用户去管理它专门的数据库。我们这里添加一个用户叫`dbuser`
{% highlight html %}
sudo adduser dbuser
{% endhighlight %}

### 修改用户postgres的密码
{% highlight html %}
sudo su - postgres
psql
\password postgres
{% endhighlight %}

### 创建数据库用户
{% highlight html %}
CREATE USER dbuser WITH PASSWORD 'password';  #! 添加dbuser，并且加上密码
CREATE DATABASE exampledb OWNER dbuser;       #! 给dbuser加上数据库
{% endhighlight %}

将exampledb数据库的所有权限都赋予dbuser，否则dbuser只能登录控制台，没有任何数据库操作权限。
{% highlight html %}
GRANT ALL PRIVILEGES ON DATABASE exampledb to dbuser;
{% endhighlight %}

### 与Rails建立关系
我们安装它们的相关依赖信息：
{% highlight html %}
sudo apt-get install libpq-dev
gem install pg
{% endhighlight %}

由于Rails项目是跑在deploy用户上的，所以目前直接访问dbuser的数据库是会有问题的，所以我们要修改一下Postgresql的配置：
{% highlight html %}
vim /etc/postgresql/9.4/main/pg_hba.conf
{% endhighlight %}
会看到在这个文件中的信息：
{% highlight html %}
# Database administrative login by Unix domain socket
local   all             postgres                                peer
# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
# IPv6 local connections:
host    all             all             ::1/128                 md5
{% endhighlight %}
由于METHOD设置为peer，所以deploy用户直接去访问dbuser的数据库信息是不可以的，所以我们得把dbuser设置为md5：
{% highlight html %}
local   all             dbuser                                md5
{% endhighlight %}

重启postgresql
{% highlight html %}
sudo service postgresql restart
{% endhighlight %}

### 数据库的备份与恢复
数据库的备份，下面的命令是mac下安装了pgAdmin3的，如果是Ubuntu，则使用数据库的用户直接调用pg_dump就可以了：
{% highlight html %}
/Applications/pgAdmin3.app/Contents/SharedSupport/pg_dump --host SERVER --port 5432 --username "USER" --password  --format custom --blobs --verbose --file "/Users/USER/Desktop/DATABASE.backup" "DATABASE"
{% endhighlight %}

数据库的恢复：
{% highlight html %}
scp /Users/USER/Desktop/DATABASE.backup USER@server.com:~/   #! 如果备份文件在本地，可以先scp上去
pg_restore --host localhost --port 5432 --username "USER" --dbname "DATABASE" --password  --verbose "/home/USER/DATABASE.backup"
{% endhighlight %}

## 安装Nodejs
因为Ubuntu源里的Nodejs版本为0.10.x，而这里我们需要安装最新的版本0.12.x：

{% highlight html %}
# Note the new setup script name for Node.js v0.12
curl -sL https://deb.nodesource.com/setup_0.12 | sudo bash -

# Then install with:
sudo apt-get install -y nodejs
{% endhighlight %}

## 安装新版本Nginx
参考：
[Install latest version of Nginx on Ubuntu](https://bjornjohansen.no/install-latest-version-of-nginx-on-ubuntu)

### 添加nginx的signing key
{% highlight html %}
curl http://nginx.org/keys/nginx_signing.key | apt-key add -
{% endhighlight %}

### 添加apt sources
{% highlight html %}
echo -e "deb http://nginx.org/packages/mainline/ubuntu/ `lsb_release -cs` nginx\ndeb-src http://nginx.org/packages/mainline/ubuntu/ `lsb_release -cs` nginx" > /etc/apt/sources.list.d/nginx.list
{% endhighlight %}

### 安装新版本的Nginx
{% highlight html %}
apt-get update
apt-get install nginx
{% endhighlight %}

### 重启Nginx
{% highlight html %}
nginx -s reload
{% endhighlight %}
