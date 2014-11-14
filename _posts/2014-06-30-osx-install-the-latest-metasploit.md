---
layout: post
title: "OSX Install The Latest Metasploit"
tags: [mac,msf]
---


把mac集成强大的渗透神器metasploit。

1. 从github上克隆Metasploit项目到本地；

2. 安装postgresql并进行配置；

3. 安装特定版本的ruby，并解决依赖；

### 从github上克隆Metasploit项目到本地
话说github真是什么都有，很多好的项目在上面都能找到，首先打开终端并输入下列命令，因为10.9.3自带了git，所以就不需要另外安装了。

    git clone https://github.com/rapid7/metasploit-framework.git /usr/local/share/metasploit-framework

克隆到本地后，将metasploit-framework/config/目录下的配置文件：``database.yml`` 添加到环境变量中（database.yml 也许不存在，直接复制database.yml.example）。

在``$HOME/.bash_profile``或者其他配置文件中添加下面这条配置：

    export MSF_DATABASE_CONFIG=/usr/local/share/metasploit-framework/config/database.yml

### 安装postgresql并进行配置
metasploit下载完后，不急着去配置，因为metasploit的默认数据库postgresql还没有装上。

你可以直接使用brew来下载并自动安装postgresql，可以运行如下命令：

    brew install postgresql --without-ossp-build

等待自动安装完毕，完成后，初始化postgresql（若出现错误删掉``/usr/local/var/postgres``，并重试）

    initdb /usr/local/var/postgres

初始化完毕后，为metasploit添加数据库用户和创建相应DB：

    createuser msf -P -h localhost    
    createdb -O msf msf -h localhost
    
（上面参数不懂了，大家可以百度一下，这里就不做过多解释了）

上面的步骤完成后，需更改metasploit的数据库连接配置，也就是第一步中的database.yml文件。

在database.yml中，将信息修改如下：

    production:  
        adapter: postgresql  
        database: msf  
        username: msf  
        password: <your password>  
        host: 127.0.0.1  
        port: 5432  
        pool: 75  
        timeout: 5
        
完成后，进入第三步！

（可自行添加alisa来简便每一次postgresql的启动）

    alias pg_start='pg_ctl -D /usr/local/var/postgres -l /usr/local/var/postgres/server.log start'  
    alias pg_stop='pg_ctl -D /usr/local/var/postgres stop'
    
### 安装特定版本的ruby，并解决依赖
说到ruby的多版本管理，这里就要用最给力的工具了rbenv（可以在github上找到，也可以使用brew直接安装）

    brew install rbenv ruby-build
    
rbenv安装完成后，在``$HOST/.bash_profile``或其他配置文件中添加如下设置：

    eval "$(rbenv init -)"
    
下面就可以通过rbenv来安装特定版本的ruby了，首先列出当前可用的ruby版本：

    rbenv install --list
    
可以看到几乎所有的ruby版本都有，这里我们选择安装ruby-1.9.3-p547（因为OS X 10.9.3 自带的ruby为2.0.0版本，在某些地方会产生问题）

    rbenv install 1.9.3-p547
    
这里可能会等一会儿，rbenv会将各个版本的ruby安装在``$HOST/.rbenv/versions/``下，完成后，将下载的版本设置为系统默认：

    rbenv rehash  
    rbenv global 1.9.3-p547
    
完成后，重新打开终端，输入 ``ruby --version`` 就可以看到当前默认的ruby版本已经设置成为1.9.3-p547

下面安装bundle来解决依赖问题：

    gem install bundle
    
安装的时候，可能会出现域名无法解析的问题，多试几次即可。

bundle安装完毕后，再次进入metasploit的主目录解决模块包的依赖：

    cd /usr/local/share/metasploit-framework  
    rbenv rehash  
    bundle install
    
（在bundle install的过程中，可能会出现某一个特定版本的模块安装失败的问题，解决方法：根据GemFiles里的版本限制，自行使用gem安装替代版本，完成后，删掉GemFiles.lock，重新运行bundle install）

依赖解决后，即可运行目录下的msfconsole启动metasploit终端控制器（已启动postgresql，不然会连不上数据库）

可以将msf命令批量ln到bin下：

    for MSF in $(ls msf*); do ln -s /usr/local/share/metasploit-framework/$MSF /usr/local/bin/$MSF;done

至此，metasploit就已经安装完毕了，enjoy this！