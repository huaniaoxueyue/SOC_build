# garylog环境安装（*离线*）

## mongodb安裝（tgz）方式
**环境参数**
```
centos7 x86
使用root用户修改配置文件：/etc/security/limits.conf
增加如下内容

* soft nproc 655360

* hard nproc 655360

* soft nofile 655360

* hard nofile 655360
```

```
安装pwgen
在一台联网的计算机上，下载pwgen的安装包。可以在pwgen官方网站（https://sourceforge.net/projects/pwgen/files/pwgen/）下载最新版本的源码包（tar.gz格式）。
tar -zxvf pwgen-x.x.x.tar.gz
./configure
make
make install
```

[mongodb]:https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel70-5.0.5.tgz "mongodbtgz包下載地址"

[es]:https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.16.3-x86_64.rpm

[graylog]:https://www.graylog.org/downloads/


安裝[mongodb]有多种方式，例如rpm、tgz等，内网建议用tgz安装

```bash
wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel70-5.0.5.tgz
```

- 将mongodb上传到tmp目录

- 默认下载路径是到用户目录下的Downloads目录，将其解压

    ```tar -zxvf mongodb-linux-x86_64-xxx.tgz```

- 将解压后的文件夹移动到/usr/local/的mongodb目录下

    ```mv mongodb-linux-x86_64-3.2.12 /usr/local/mongodb```

- 进入mongdb目录在此目录下创建data/db 和data/logs文件夹
    ```cd /usr/local/mongodb && mkdir -p data/db && mkdir -p data/logs```

- 配置系统文件profile

    ```sudo vi /etc/profile```
    末尾插入下列内容：

    ```
    export PATH=$PATH:/usr/local/mongodb/bin
    ```
    应用环境变量:
    ```source /etc/profile```


- 配置**mongo.conf**
    ```
    # 数据库数据存放目录
    dbpath = /usr/local/mongodb/data/db
    # 数据库日志存放目录
    logpath = /usr/local/mongodb/data/logs/mongodb.log
    # 以追加的方式记录日志
    logappend = true
    # 端口号
    port = 27017
    # 以后台方式启动
    fork = true
    # 允许外部访问，127.0.0.1仅本机可访问
    bind_ip = 0.0.0.0
    #增加认证
    #auth = true
    ```

- 启动时带上配置文件
    ```
    ./mongod -f ./mongod.conf
    netstat -antpl | grep mongod    #查看mongod是否正常启动
    ```

- 增加mongodb密码：
    打开 MongoDB shell
    连接到要添加用户的数据库：
    ```use mydatabase```
    使用 db.createUser() 方法创建新用户和密码。例如，创建一个名为 myuser，密码为 mypassword 的用户，具有读取和写入数据库的权限，可以使用以下命令
    ```mongodb
    db.createUser({
        user: "myuser",
        pwd: "mypassword",
        roles: [
            { role: "readWrite", db: "mydatabase" }
        ]
    })
    ```
- 打开登录需认证的参数,开启auth=true
  ```
  ./mongod -f ./mongod.conf --shutdown
  vi mongo.conf
  ./mongod -f ./mongod.conf
  ps -aux | grep mongod
  ```

修改mongodb密码方法：
切换至mongo的bin目录下，登录mongo
登陆成功后，切换至admin表 （mongodb的所有用户都会存储在admin表中，所以需要切换至admin表再进行用户的修改）
```
use admin;
db.changeUserPassword('用户名','新密码');
db.auth('用户名','新密码');
```

## 安装es
**这里通过rpm方式安装es**

- 下载安装包
    ```
    https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.16.3-x86_64.rpm
    ```

- 安装es
    ```
    rpm -ivh elasticsearch-7.16.3-x86_64.rpm
    ```

- 启动es
    ```
    systemctl start elasticsearch
    ```
- 验证 Elasticsearch 是否启动成功：
    ```
    curl http://localhost:9200/
    ```
- 返回以下内容，则安装成功
    ```
    {
    "name" : "hostname",
    "cluster_name" : "elasticsearch",
    "cluster_uuid" : "GrsQ2...",
    "version" : {
        "number" : "7.15.0",
        "build_flavor" : "default",
        "build_type" : "rpm",
        "build_hash" : "..."
    },
    "tagline" : "You Know, for Search"
    }
    ```
如果启动失败，可以查看 Elasticsearch 的日志文件，路径为 /var/log/elasticsearch/。
- 设置 Elasticsearch 开机自启动：
    ```
    systemctl enable elasticsearch
    ```

**为了避免 Elasticsearch 的未授权访问漏洞，可以采取以下措施：**

设置访问控制列表（ACL）：可以通过配置 Elasticsearch 的 elasticsearch.yml 文件，设置允许访问 Elasticsearch 的 IP 地址范围。例如，可以添加以下内容：

    ```
    http.host: 0.0.0.0
    transport.host: 127.0.0.1
    network.host: 0.0.0.0

    # 添加下面两行
    http.cors.enabled: true
    http.cors.allow-origin: "*"

    # 添加下面三行
    xpack.security.enabled: true
    xpack.security.transport.ssl.enabled: true
    xpack.security.transport.ssl.verification_mode: certificate
    ```
上面的配置允许任何 IP 地址访问 Elasticsearch，并开启了跨域资源共享（CORS）和安全认证。

启用安全认证：可以通过 Elasticsearch 的 X-Pack 插件启用安全认证，实现基于用户名和密码的访问控制。启用 X-Pack 插件后，需要创建超级用户并设置密码，然后为每个用户创建角色，并授予不同的权限。例如，可以使用以下命令为超级用户创建密码：
```
sudo /usr/share/elasticsearch/bin/elasticsearch-setup-passwords interactive
```
运行上面的命令后，按照提示输入密码并确认即可。然后，可以使用 elasticsearch-users 工具创建其他用户和角色，例如：
```
sudo /usr/share/elasticsearch/bin/elasticsearch-users useradd myuser -r superuser
```
上面的命令创建了一个名为 myuser 的用户，并将其赋予超级用户角色。

使用网络安全组：如果 Elasticsearch 运行在云服务器上，可以通过设置网络安全组，限制仅允许指定的 IP 地址访问 Elasticsearch。这可以确保 Elasticsearch 不会被未经授权的访问所利用。

综上所述，可以通过设置访问控制列表、启用安全认证和使用网络安全组等措施，有效避免 Elasticsearch 的未授权访问漏洞。

## 安装graylog
安裝[graylog]有多种方式，例如docker、rpm、tgz等，内网建议用tgz

- 安装

    ```
    mkdir /usr/local/graylog
    cp graylog-server-4.3.0.tgz /usr/local/graylog
    cd /usr/local/graylog
    tar -zxvf graylog-server-4.3.0.tgz
    ```
- 复制graylog配置文件
    ```
    cp graylog.conf.example /etc/graylog/server/server.conf 
    ```

- 更改graylog配置文件
  - 创建password_secret的值

    ```
    pwgen -N 1 -s 96
    ```
  - 创建graylog的web登录密码配置root_password_sha2
    ```
    echo -n "Enter Password: " && head -1 </dev/stdin | tr -d '\n' | sha256sum | cut -d" " -f1
    ```
  - 更改配置文件
    ```
    password_secret = 你生成的值
    root_password_sha2 = 你生成的值
    root_timezone = Asia/Shanghai
    http_bind_address = 0.0.0.0:9000
    elasticsearch_hosts = http://127.0.0.1:9200
    mongodb_uri = mongodb://admin:admin@localhost:27017/graylog?authSource=admin
    ```

- 启动graylog
    ```
    ./graylogctl status
    ```

- 如果graylog未正常启动
  如果执行 ./graylogctl start 命令后 graylog 并未启动，可能需要先检查一下 graylog 的日志信息，以了解具体出错原因。graylog 默认的日志文件位于 /var/log/graylog-server/ 目录下，你可以在该目录下查看 graylog 的日志文件，例如
```
tail -f /var/log/graylog-server/server.log
```