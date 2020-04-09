---
title: KONG部署
date: 2020-04-09 20:40:00
author: DemonLee
img: /medias/banner/kong&docker.png
coverImg: /medias/banner/kong&docker.png
top: true
cover: true
toc: true
mathjax: false
summary: 
  总结一下上一份工作的有关于API 网关KONG的内容，简单做一下回顾和总结
tags:
  - Hexo
  - Github
  - 博客
categories:
  - API 网关KONG
password:
---

# KONG Docker 部署方式


## 前期准备
#0 运行容器不退出
docker run -itd --rm  kong:1.3.0-ubuntu /bin/bash -c "while true;do echo hello docker;sleep 1;done"
#1 重启docker
systemctl daemon-reload && systemctl restart docker
postgres 镜像： demonlee/postgres:9.6
kong 1.3版本镜像： demonlee/kong:1.3
mysql镜像： demonlee/mysql:5.5

## postgres + kong
#1 创建本地卷pgdata
docker volume create pgdate
#2 创建kong-net-postgres
docker network create kong-net-postgres
#3 创建 pg-kong 容器
docker run -d --name ps-kong  \ 
        --network=kong-net-postgres  \ 
        -v pgdate:/var/postgres/data \ 
        -p 5432:5432 \ 
        -e "POSTGRES_U SER=kong" \ 
        -e "POSTGRES_DB=kong" \ 
        -e "POSTGRES_PASSWORD=kong" \ 
        demonlee/postgres:9.6

#4 初始化kong在pg中的数据库
docker run --rm \
     --network=kong-net-postgres \
     -e "KONG_DATABASE=postgres" \
     -e "KONG_PG_HOST=ps-kong"   \
     -e "KONG_PG_PASSWORD=kong"  \
     demonlee/kong:1.3 kong migrations bootstrap

#5 启动kong容器

## 1 开放管理端口
docker run -d --name kong-postgres  \
     --network=kong-net-postgres    \
     -e "KONG_DATABASE=postgres"    \
     -e "KONG_PG_HOST=ps-kong"      \
     -e "KONG_PG_PASSWORD=kong"  \
     -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
     -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
     -e "KONG_PROXY_ERROR_LOG=/dev/stderr"  \
     -e "KONG_ADMIN_ERROR_LOG=/dev/stderr"  \
     -e "KONG_ADMIN_LISTEN=0.0.0.0:18001, 0.0.0.0:18444 ssl" \
     -e "KONG_PROXY_LISTEN=0.0.0.0:18000, 0.0.0.0:18443 ssl" \
     -p 18000:18000 \
     -p 18443:18443 \
     -p 18001:18001 \
     -p 18444:18444 \
     demonlee/kong:1.3

## 2关闭管理端口
docker run -d --name kong-postgres  \
     --network=kong-net-postgres    \
     -e "KONG_DATABASE=postgres"    \
     -e "KONG_PG_HOST=ps-kong"      \
     -e "KONG_PG_PASSWORD=kong"  \
     -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
     -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
     -e "KONG_PROXY_ERROR_LOG=/dev/stderr"  \
     -e "KONG_ADMIN_ERROR_LOG=/dev/stderr"  \
     -e "KONG_ADMIN_LISTEN=0.0.0.0:18001, 0.0.0.0:18444 ssl" \
     -e "KONG_PROXY_LISTEN=0.0.0.0:18000, 0.0.0.0:18443 ssl" \
     -p 18000:18000 \
     -p 18443:18443 \
     -p 127.0.0.1:18001:18001 \
     -p 127.0.0.1:18444:18444 \
     demonlee/kong:1.3
	 
## mysql + kong
#1 创建本地卷mysqldata
docker volume create mysqldata
#2 创建kong-net-mysql
docker network create kong-net-mysql
#3 创建 mysql-kong 容器
docker run -d --name mysql-kong \
               --network=kong-net-mysql \
			   -v mysqldata:/var/mysql/data \
               -p 3306:3306 \
               -e "MYSQL_USER=kong" \
               -e "MYSQL_DB=kong" \
               -e "MYSQL_PASSWORD=kong" \
               demonlee/mysql:5.5

#4 初始化kong在mysql中的数据库
docker run --rm \
     --network=kong-net-mysql \
     -e "KONG_DATABASE=mysql" \
     -e "KONG_MYSQL_HOST=mysql-kong" \
     -e "KONG_MYSQL_PASSWORD=kong" \
     demonlee/kong:1.3 kong migrations bootstrap

#5 启动kong容器
docker run -d --name kong-mysql \
     --network=kong-net-mysql \
     -e "KONG_DATABASE=mysql" \
     -e "KONG_MYSQL_HOST=mysql-kong" \
     -e "KONG_MYSQL_PASSWORD=kong" \
     -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
     -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
     -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
     -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
     -e "KONG_ADMIN_LISTEN=0.0.0.0:28001, 0.0.0.0:28444 ssl" \
	 -e "KONG_PROXY_LISTEN=0.0.0.0:28000, 0.0.0.0:28443 ssl" \
     -p 28000:28000 \
     -p 28443:28443 \
     -p 127.0.0.1:28001:28001 \
     -p 127.0.0.1:28444:28444 \
     demonlee/kong:1.3
	 