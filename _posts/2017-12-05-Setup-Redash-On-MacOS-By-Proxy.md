---
layout: post
title: 在Mac系统上网络代理内安装Redash
---


## 前言

  redash是一款企业数据视图化工具。技术栈为**python+flask+angular+regresql+redis**。python包管理工具pip，应用虚拟化docker。为了解决python包冲突使用了virtualenv。

  横向比较了几款同类型工具，最终选择Redash来做进一步了解。finereport呼声特别高，由于商业化文档插件支持比较好，据说论坛反映的问题会有回应甚至在新版中更新，功能据说根据国内企业环境优化了，但是由于商业化的原因直接pass掉了。superset可视化的选项比redash多，对非技术人员更友好些，但是redash的代码更规范。最后选择了Redash。具体过程就不解释了。
    
  由于工作的网络环境在代理下，所以一直以来在终端中需要网络的命令都会异常。通过分析发现所使用的代理只是基本类型的，绝大部分终端应用都会支持该方式的认证。一般只需要设置全局变量`http_proxy`和`https_proxy`就能够通过代理。在尝试解决代理问题过程中发现了很有用的工具cntlm。
  
> proxy: 网络代理这里的涉及到basic和ntlm。basic只是用base64编码一下，ntlm更加安全，网络上没有密码. [ntlm百度百科](https://baike.baidu.com/item/NTLM/6371298)

> cntlm: 该工具可以用来完成ntlm认证，过程为：在本地创建一个无认证的代理服务器，该代理服务器根据配置文件(/etc/cntlm.conf)中的账号和密码（也可以用参数传入，见manual）来连接原来的cntlm代理服务器，终端应用配置本地的代理就能通过本地代理来认证cntlm代理服务器进而访问网络。
> 
特别: 本教程是笔者按照回忆和之前留下的记录来编写的，并没有复现验证，所以可能存在一些谬误。另外，一些命令需要修改一下参数，比如benjmin作为本地用户被多次用到，在其他环境需要适当修改。

## 安装
  
  Redash的安装对ubuntu的支持比较好，有直接的安装脚本。而macos可以通过docker和pip来安装环境，总体也算方便。笔者由于cntlm连接异常无法为docker提供网络环境，所以regresql和redis是安装到本地的。macos系统安装过程如下：

 0. 配置网络代理
 1. 安装postgresql和redis软件。
 2. 使用pip安装virtualenv和redash的python环境。
 3. 准备并开启redash。

系统环境：

```
macox Sierra v10.12.3
Python 2.7.10
```

### 配置全局变量`http_proxy`和`https_prox`, 为安装工具配置好代理

```bash
export http_proxy="http://<username>:<password>@10.191.131.3:3128"
export https_proxy="http://<username>:<password>@10.191.131.3:3128"
```
### 安装postgresql
[下载官方地址 v2.1.1 dmg](https://github.com/PostgresApp/PostgresApp/releases/download/v2.1.1/Postgres-2.1.1.dmg)

 *安装完成需要配置bin目录到path*
 
### 安装redis
[下载源码v4.0.5](http://download.redis.io/releases/redis-4.0.5.tar.gz)

*也可以使用Homebrew安装*

```bash
brew intall redis
```

### 使用pip安装virtualenv并为redash创建虚拟环境并激活
略
### 在虚拟环境中安装python依赖库。

```bash
# 设置环境变量
redash_home=/path/to/redash
REDASH_BRANCH="${REDASH_BRANCH:-master}"

# 根据requirements文件安装依赖库
cd $redash_home
sudo -E pip -r requirements_dev.txt -r requirements_all_ds.txt 
```
安装依赖库时需要配置postgresql和redis的bin目录到path。

```bash
export PATH=$PATH:/Applications/Postgres.app/Contents/Versions/9.6/bin
```
笔者还遇到安装xcode tool的情况，reinstall就可以。

### 创建redash环境变量.env

```bash
# benjamin is a local user
sudo -E -u benjamin curl "https://raw.githubusercontent.com/getredash/redash/${REDASH_BRANCH}/setup/ubuntu/files/env" -o $redash_home/.env
COOKIE_SECRET=$(sf-pwgen -a random -c 1 -l 32)
echo "export REDASH_COOKIE_SECRET=$COOKIE_SECRET" >> $redash_home/.env
```
需要安装sf-pwgen
> sf-pwgen: 随机数生成工具。可以通过brew安装，命令*brew install sf-pwgen*

### 创建数据库

```bash
# benjamin is a local user
sudo -u benjamin createuser redash --no-superuser --no-createdb --no-createrole
sudo -u benjamin createdb redash --owner=redash

cd $redash_home
sudo -u benjamin bin/run ./manage.py database create_tables
```
### 准备前端js

```bash
npm install
npm run dev
```
### 运行postgresql数据库服务器
打开postgresql.app

### 运行redis服务器

```bash
setup/amazon_linux/files/redis_init start
```
### 开启celery
```bash
bin/run /Users/benjamin/env_redash/bin/celery worker --app=redash.worker --beat -Qqueries,celery,scheduled_queries
```
创建用于抓数据的队列

### 运行redash

```bash
bin/run ./manage.py runserver --debugger
```
默认在本地5000端口开启服务。`--help`参数查看所有runserver命令的参数。

