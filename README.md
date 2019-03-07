# FFXIVBOT Docker

[![Build Status](https://img.shields.io/docker/cloud/build/bluefissure/ffxivbot.svg)](https://cloud.docker.com/repository/docker/bluefissure/ffxivbot) [![Downloads](https://img.shields.io/docker/pulls/bluefissure/ffxivbot.svg)](https://cloud.docker.com/repository/docker/bluefissure/ffxivbot)  [![Size](https://img.shields.io/microbadger/image-size/bluefissure/ffxivbot.svg)](https://cloud.docker.com/repository/docker/bluefissure/ffxivbot)

[![CodeFactor](https://www.codefactor.io/repository/github/bluefissure/ffxivbot/badge/master)](https://www.codefactor.io/repository/github/bluefissure/ffxivbot/overview/master) [![license](https://img.shields.io/badge/license-GPL-blue.svg)](https://github.com/Bluefissure/FFXIVBOT/blob/master/LICENSE)



***该镜像由Linux内核构建，无法在Windows服务器上运行，只能在Windows物理机中构建***



## 安装Docker

### Linux

#### docker-ce

```bash
curl -sSL https://get.docker.com/ | sh 
```

如果不是root安装，安装过程可能要输入root密码

如果`container.io`的安装有问题，可以通过先`sudo apt-get remove docker-ce`，再`sudo apt-get remove runc`，再重试上述命令尝试解决

#### docker-compose

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.23.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```

### Windows

#### docker-ce

https://docs.docker.com/docker-for-windows/

#### docker-compose

https://docs.docker.com/compose/install/

## 下载Compose配置并启动

***Linux***

```bash
wget https://raw.githubusercontent.com/Bluefissure/FFXIVBOT/docker/release/docker-compose.yml
docker-compose pull
docker-compose up
```

***Windows***

```powershell
wget https://raw.githubusercontent.com/Bluefissure/FFXIVBOT/docker/release/docker-compose.yml -OutFile docker-compose.yml
docker-compose pull
docker-compose up
```

然后服务就启动了，可以通过IP:8000端口访问，如果需要更改端口请更改`docker-compose.yml`文件。

然而，应该会报错，因为数据库还没有初始化。

## 数据库初始化，导入数据

### 数据库初始化

之前的docker不要关，用以下命令进入docker：

```bash
docker exec -t -i ffxivbot-web /bin/bash
```

之后依次运行如下代码初始化数据库

```bash
python manage.py makemigrations
python manage.py migrate
```

***请注意，如果代码更新，需要在`docker-compose pull`之后重新运行以上两条命令。***

然后输入以下cron爬取微博：

***Linux***

```bash
nohup python /FFXIVBOT/utils/crawl_wb.py >> /var/log/weibo_cron.log &
```

***Windows***

```powershell
python /FFXIVBOT/utils/crawl_wb.py 
```

通过以下代码创建超级管理员：

```bash
python manage.py createsuperuser
```

之后Ctrl+D退出docker，并通过IP:8000端口访问即可

### 启动RBQ

来到`/FFXIVBOT/ffixvbot`目录，运行：

```bash
nohup python /FFXIVBOT/ffixvbot/pika_rabbit.py 2>/dev/null &
```

如果要多个RBQ消费者，多运行几次上面的代码即可。

<del>这一步我还没测试，但是可以和巨硬学习把用户当作测试部门。</del>

### 数据库同步

结构导入了，但是数据库还没有数据，机器人可以通过网页添加，但是诸如/dps一类的功能都需要同步boss的ID之类的数据才能正常使用

首先下载dump的数据：

***Linux***

```bash
wget https://raw.githubusercontent.com/Bluefissure/FFXIVBOT/docker/release/FFXIV_DEV.sql 
sudo mv FFXIV_DEV.sql docker/mysql-dump/
```

***Windows***

```
wget https://raw.githubusercontent.com/Bluefissure/FFXIVBOT/docker/release/FFXIV_DEV.sql -OutFile FFXIV_DEV.sql
mv FFXIV_DEV.sql docker\mysql-dump\
```

然后我们进入db的docker：

```bash
docker exec -t -i ffxivbot-db mysql -uroot -proot
```

然后会出现以`mysql>`开头的命令行选择数据库并导入数据

***Linux***

```mysql
use FFXIV_DEV;
source /mysql-dump/FFXIV_DEV.sql;
```

***Windows***

```mysql
use FFXIV_DEV;
source ./mysql-dump/FFXIV_DEV.sql;
```

可能由于一些外键的存在，要导入sql文件多次才能导入全部数据。

导入后，重启docker即可。
