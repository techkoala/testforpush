# 搭建CouchDB数据库

> 记录一下 CouchDB 搭建流程，用于同步 MoonFM、Obsidian



很早之前就购买了 MoonFM，但是鉴于没有自带同步服务，所以一直使用 Spotify 听播客。现在终于忍不了这种音乐和播客混在一起，乱七八糟的感觉了，因此抽空研究了 MoonFM 使用的 Couchdb 数据库同步方案，实现了自建同步服务。同时也刚好一并将 Obsidian 的同步服务从[Remotely save](https://github.com/remotely-save/remotely-save)迁移到了[obsidian-livesync](https://github.com/vrtmrz/obsidian-livesync)，实现了更好的同步体验！

## Docker 安装 CouchDB

首先需要创建一个 data 文件夹和一个 local.ini 用于 docker 内部文件夹和配置文件的映射。其中 local.ini 如下：

```yaml
[couchdb]
single_node=true

[chttpd_auth]
require_valid_user = true
authentication_redirect = /_utils/session.html

[httpd]
WWW-Authenticate = Basic realm="couchdb"
enable_cors = true

[cors]
origins = capacitor://localhost,http://localhost
credentials = true
headers = accept, authorization, content-type, origin, referer
methods = GET, PUT, POST, HEAD, DELETE
max_age = 3600

[couch_peruser]
database_prefix = userdb_
delete_dbs = false
enable = true

```

docker-compose 文件如下，需要自己修改一下映射端口，账号密码等：

```yaml
version: "3"
services:
couchdb:
ports:
  - "5984:5984"
volumes:
  - "./data:/opt/couchdb/data"
  - "./local.ini:/opt/couchdb/etc/local.ini"
environment:
  - COUCHDB_USER=管理后台账户名
  - COUCHDB_PASSWORD=管理后台密码
restart: always
network_mode: bridge
container_name: couchdb
image: couchdb
```

然后使用`docker-compose up -d`启动 docker

## 数据库管理

通过`http://IP地址:端口号/_utils/`访问数据口后台，账号密码就是上一步自己定义的密码。

页面中就可以创建并管理自己的数据库和用户，但是我更习惯使用命令行进行管理：

以 MoonFM 同步数据库创建为例，依次执行下面的命令即可：

- 创建用户

  ```
  curl -X PUT http://管理后台账户名:管理后台密码@IP地址:端口号/_users/org.couchdb.user:用户账号名
  -H "Accept: application/json"
  -H "Content-Type: application/json"
  -d '{"name": "用户账号名", "password": "用户密码", "roles": [], "type": "user"}'
  ```
- 创建数据库

```
curl -X PUT http://管理后台账户名:管理后台密码@IP地址:端口号/数据库名
```

- 添加用户到数据库

  ```
  curl -X PUT http://管理后台账户名:管理后台密码@IP地址:端口号/数据库名/_security
  -H "Accept: application/json"
  -H "Content-Type: application/json"
  -d '{"admins": {"names": [], "roles": []}, "members": {"names": ["用户账号名"], "roles": []}}'
  ```

获得的同步链接如下：

```
http://用户账号名:用户账号密码@IP地址:端口号/数据库名
```

> 这里需要注意区分 curl 参数填入的管理后台账户名密码和数据库用户名密码

此后还可以进行反向代理等操作，这里可以参考[使用 Nginx 实现多服务复用端口](https://www.techkoala.net/nginx_port_reuse/)

## fly.io 搭建免费 CouchDB 数据库

未完待续

