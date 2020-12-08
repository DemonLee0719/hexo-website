---
title: KONG适配mysql
date: 2020-05-03 19:27:00
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
# KONG 1.3适配mysql5.6

## What?
API网关KONG是一个开源项目，详见 [KONG官方文档](https://docs.konghq.com/)

## Why?
api网关KONG数据库目前只支持postgres和Cassandra，还有无数据库dbless。对于mysql没有支持，由于公司大佬要求需要统一数据库mysql，所以只能基于kong1.3支持mysql。

## How?
mysql适配仅限于kong1.3版本。
主要代码部分在kong/db/strategies/mysql文件下，其中connector.lua文件主要适配，支持kong使用mysql。其中很多代码是借鉴postgres，若你下载我们适配的kong-mysql-support1.3，你会在mysql下看到很多postgres的影子，在kong/tools文件下还有借鉴章亦春大佬的mysql.lua，openresty中连接mysql的一个工具，有一些细微改动。
[GitHub代码地址]( https://github.com/DemonLee0719/kong-mysql-support-1.3.0 )

### kong代码导读
kong网关很多优秀的特性，如可扩展性，模块化，可移植性，插件机制等等这里不再详细描述，请参考官方文档 [KONG官方文档](https://docs.konghq.com/)，这里不再搬运。

#### kong项目代码结构
kong/api：KONG主要对外的api管理，如service，routes等等，可以通过restful接口用于注册管理api等。定义模块的类型和格式，初度kong源码会有一些迷，比如，kong的关于路由route的接口是怎么实现的，就在kong/api下由模块统一管理。
kong/cluster_events：该目录下，是对于kong的event事件做统一同步，若你是通过k8s或者其他可节点方式部署，多节点信息同步实现在该文件下实现。可以在配置项db_update_frequency = 5，官方默认配置为5秒同步。当其他节点的信息有变动时，会把改动的数据写入到数据库中，有一个cluster_events表，当kong的实体，如service，route通过restful接口变动时，会将变动数据写入，其他节点会每5秒拉去数据，同步。
添加文件 kong/cluster_events/strategies/mysql.lua 

```lua
local utils  = require "kong.tools.utils"
local mysql = require "kong.tools.mysql"
local pgmoon = require "pgmoon"


local max          = math.max
local fmt          = string.format
local null         = ngx.null
local concat       = table.concat
local setmetatable = setmetatable
local new_tab
do
  local ok
  ok, new_tab = pcall(require, "table.new")
  if not ok then
    new_tab = function(narr, nrec) return {} end
  end
end


local INSERT_QUERY = [[
INSERT INTO cluster_events(`id`, `node_id`, at, nbf, expire_at, channel, data)
 VALUES(%s, %s, FROM_UNIXTIME(%f), FROM_UNIXTIME(%s), FROM_UNIXTIME(%s), %s, %s)
]]

local SELECT_INTERVAL_QUERY = [[
SELECT `id`, `node_id`, channel, data, UNIX_TIMESTAMP(at) as at, UNIX_TIMESTAMP(nbf) as  nbf
FROM cluster_events
WHERE channel IN (%s)
  AND at >  FROM_UNIXTIME(%f)
  AND at <= FROM_UNIXTIME(%f)
]]


local _M = {}
local mt = { __index = _M }


function _M.new(db, page_size, event_ttl)
  local self  = {
    db        = db.connector,
    --page_size = page_size,
    event_ttl = event_ttl,
  }

  return setmetatable(self, mt)
end


function _M.should_use_polling()
  return true
end

function _M:insert(node_id, channel, at, data, nbf)
  local expire_at = max(at + self.event_ttl, at)

  if not nbf then
    nbf = "NULL"
  end

  local my_id      = ngx.quote_sql_str(utils.uuid())
  local my_node_id = ngx.quote_sql_str(node_id)
  local my_channel = ngx.quote_sql_str(channel)
  local my_data    = ngx.quote_sql_str(data)

  local q = fmt(INSERT_QUERY, my_id, my_node_id, at, nbf, expire_at,
    my_channel, my_data)

  local res, err = self.db:query(q)
  if not res then
    return nil, "could not insert invalidation row: " .. err
  end

  return true
end


function _M:select_interval(channels, min_at, max_at)
  local n_chans = #channels
  local my_channels = new_tab(n_chans, 0)

  for i = 1, n_chans do
    my_channels[i] = pgmoon.Postgres.escape_literal(nil, channels[i])
  end

  local q = fmt(SELECT_INTERVAL_QUERY, concat(my_channels, ","), min_at,
    max_at)

  local ran

  -- TODO: implement pagination for this strategy as
  -- well.
  --
  -- we need to behave like lua-cassandra's iteration:
  -- provide an iterator that enters the loop, with a
  -- page = 0 argument if there is no first page, and a
  -- page = 1 argument with the fetched rows elsewise

  return function(_, p_rows)
    if ran then
      return nil
    end

    local res, err = self.db:query(q)
    if not res then
      return nil, err
    end

    local len = #res
    for i = 1, len do
      local row = res[i]
      if row.nbf == null then
        row.nbf = nil
      end
    end

    local page = len > 0 and 1 or 0

    ran = true

    return res, err, page
  end
end


function _M:truncate_events()
  return self.db:query("TRUNCATE cluster_events")
end


return _M
```

kong/cmd： 当前目录下是kong相关命令是实现，这里不再详细描述。
相关命令参考kong/cmd/init.lua
```lua
require("kong.globalpatches")({cli = true})

math.randomseed() -- Generate PRNG seed

local pl_app = require "pl.lapp"
local log = require "kong.cmd.utils.log"

local options = [[
 --v              verbose
 --vv             debug
]]

local cmds_arr = {}
local cmds = {
  start = true,
  stop = true,
  quit = true,
  restart = true,
  reload = true,
  health = true,
  check = true,
  prepare = true,
  migrations = true,
  version = true,
  config = true,
  roar = true
}
```

kong/db：该目录下是kong对数据库的支持，适配mysql中的主要工作在此，需要对很多数据层做支持，还有sql语句改写，以及相关策略改动
kong/db/migrations/core下是在执行kong migrations bootstraps时会初始化数据表，当前提是当前连接的数据库中有kong这个一个数据库，不然会报错。
kong/db/strategies下是度数据库选择策略的适配，官方目前有Cassandra，postgres,off，这里对mysql做了适配。可放心使用。

kong/pdk： 该目录下的文件是对kong有开发需求的开发人员提供特定的pdk方法，若想要开发相关插件，需要关注这个一块。话说，kong官方的插件很多还是需要自己二次开发滴！！！

kong/plugins：该目录下放着需要的一些插件，可以在当前目录下开发自己自定义的插件。请参考kong插件开发规范开发插件，比较容易。插件的使用导入需要在kong/constants.lua文件中体现
kong/constants.lua部分代码
```lua
local plugins = {
  "jwt",
  "acl",
  "correlation-id",
  "cors",
  "oauth2",
  "tcp-log",
  "udp-log",
  "file-log",
  "http-log",
  "key-auth",
  "hmac-auth",
  "basic-auth",
  "ip-restriction",
  --"request-transformer",
  "response-transformer",
  "request-size-limiting",
  "rate-limiting",
  "response-ratelimiting",
  "syslog",
  "loggly",
  "datadog",
  "ldap-auth",
  "statsd",
  "bot-detection",
  "aws-lambda",
  "request-termination",
  -- external plugins
  --"azure-functions",
  --"kubernetes-sidecar-injector",
  --"zipkin",
  --"pre-function",
  --"post-function",
  --"prometheus",
  --"proxy-cache",
  --"session",
}
```

kong/resty：当前目录是加载相关openresty 的某些文件，没有详细研究

kong/runloop：此目录下的文件上kong主要和nginx交互的文件目录，相关api注册还有路由转发，环平衡，健康检查等相关在这。

kong/templates：当前目录下是配置一些nginx的文件，通过导入，直接配置到nginx.conf文件。

kong/tools：工具类目录

kong/route.lua：该lua主要服务kong的路由转发还有就是一些初始化路由检查，路由加载等。

更多需要详细阅读源码，这里只浅显解读一下。

## 前期准备
###  0运行容器不退出，调试需要
```shell
docker run -itd --rm  kong:1.3 /bin/bash -c "while true;do echo hello docker;sleep 1;done"
```
###  1重启docker
```shell
systemctl daemon-reload && systemctl restart docker
```
postgres 镜像： postgres:9.6 （自行dockerhub下载）
kong 1.3版本镜像： kong:1.3 （自行dockerhub下载）
mysql镜像： mysql:5.6          （自行dockerhub下载）
kong-mysql适配镜像： kong-mysql-support:1.3 
这个dockerfile比较简单粗暴，代码仓库中很多插件都屏蔽了，若读者想要添加一些我在代码中屏蔽的插件，需要自行适配mysql
```dockerfile
FROM kong:1.3
RUN rm -rf /usr/local/share/lua/5.1/kong
COPY kong /usr/local/share/lua/5.1/kong
```

## postgres + kong 官方docker部署方式
#1 创建本地卷pgdata
```bash
docker volume create pgdate
```
#2 创建kong-net-postgres
```bash
docker network create kong-net-postgres
```
#3 创建 pg-kong 容器
```bash
docker run -d --name pg-kong  \ 
        --network=kong-net-postgres  \ 
        -v pgdate:/var/postgres/data \ 
        -p 5432:5432 \ 
        -e "POSTGRES_U SER=kong" \ 
        -e "POSTGRES_DB=kong" \ 
        -e "POSTGRES_PASSWORD=kong" \ 
        postgres:9.6
```
#4 初始化kong在pg中的数据库
```bash
docker run --rm \
     --network=kong-net-postgres \
     -e "KONG_DATABASE=postgres" \
     -e "KONG_PG_HOST=ps-kong"   \
     -e "KONG_PG_PASSWORD=kong"  \
     kong:1.3 kong migrations bootstrap --vv 
```
#5 启动kong容器
##1 开放管理端口，为了好调试，开放所有端口，生成环境不建议这么做。
```bash
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
     kong:1.3
```
##2关闭管理端口，建议生产环境配置
```bash
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
     kong:1.3
```

## mysql + kong  mysql适配版
#1 创建本地卷mysqldata
docker volume create mysqldata
#2 创建kong-net-mysql
docker network create kong-net-mysql
#3 创建 mysql-kong 容器
```bash
docker run -d --name mysql-kong \
               --network=kong-net-mysql \
               -v mysqldata:/var/mysql/data \
               -p 3306:3306 \
               -e "MYSQL_USER=kong" \
               -e "MYSQL_DB=kong" \
               -e "MYSQL_PASSWORD=kong" \
               demonlee/mysql:5.5
```
#4 初始化kong在mysql中的数据库
```bash
docker run --rm \
     --network=kong-net-mysql \
     -e "KONG_DATABASE=mysql" \
     -e "KONG_MYSQL_HOST=mysql-kong" \
     -e "KONG_MYSQL_PASSWORD=kong" \
     demonlee/kong:1.3 kong migrations bootstrap
```
#5 启动kong容器
```bash
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
     -p 28001:28001 \
     -p 28444:28444 \
     demonlee/kong:1.3
```