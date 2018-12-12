CentOS 7 安装文档
--------------------------

说明
~~~~~~~
-  # 开头的行表示注释
-  > 开头的行表示需要在 mysql 中执行
-  $ 开头的行表示需要执行的命令

本文档适用于有一定web运维经验的管理员或者工程师,文中不会对安装的软件做过多的解释,仅对需要执行的内容注部分注释,更详细的内容请参考一步一步安装。

安装过程中遇到问题可参考 `安装过程中常见的问题 <faq_install.html>`_

环境
~~~~~~~

-  系统: CentOS 7
-  IP: 192.168.244.144
-  目录: /opt
-  数据库: mariadb
-  代理: nginx

开始安装
~~~~~~~~~~~~

.. code-block:: shell

    $ yum update -y

    # 防火墙 与 selinux 设置说明,如果已经关闭了 防火墙 和 Selinux 的用户请跳过设置
    $ systemctl start firewalld
    $ firewall-cmd --zone=public --add-port=80/tcp --permanent  # nginx 端口
    $ firewall-cmd --zone=public --add-port=2222/tcp --permanent  # 用户SSH登录端口 coco
      --permanent  永久生效,没有此参数重启后失效

    $ firewall-cmd --reload  # 重新载入规则

    $ setenforce 0
    $ sed -i "s/enforcing/disabled/g" `grep enforcing -rl /etc/selinux/config`

    # 修改字符集,否则可能报 input/output error的问题,因为日志里打印了中文
    $ localedef -c -f UTF-8 -i zh_CN zh_CN.UTF-8
    $ export LC_ALL=zh_CN.UTF-8
    $ echo 'LANG="zh_CN.UTF-8"' > /etc/locale.conf

    # 安装依赖包
    $ yum -y install wget gcc epel-release git

    # 安装 Redis, Jumpserver 使用 Redis 做 cache 和 celery broke
    $ yum -y install redis
    $ systemctl enable redis
    $ systemctl start redis

    # 安装 MySQL,如果不使用 Mysql 可以跳过相关 Mysql 安装和配置,支持sqlite3, mysql, postgres等
    $ yum -y install mariadb mariadb-devel mariadb-server # centos7下叫mariadb,用法与mysql一致
    $ systemctl enable mariadb
    $ systemctl start mariadb
    # 创建数据库 Jumpserver 并授权
    $ mysql -uroot
    > create database jumpserver default charset 'utf8';
    > grant all on jumpserver.* to 'jumpserver'@'127.0.0.1' identified by 'weakPassword';
    > flush privileges;
    > quit

    # 安装 Nginx ,用作代理服务器整合 Jumpserver 与各个组件
    $ vi /etc/yum.repos.d/nginx.repo

    [nginx]
    name=nginx repo
    baseurl=http://nginx.org/packages/centos/7/$basearch/
    gpgcheck=0
    enabled=1

    $ yum -y install nginx
    $ systemctl enable nginx

    # 安装 Python3.6
    $ yum -y install python36 python36-devel

    # 配置并载入 Python3 虚拟环境
    $ cd /opt
    $ python3.6 -m venv py3  # py3 为虚拟环境名称,可自定义
    $ source /opt/py3/bin/activate  # 退出虚拟环境可以使用 deactivate 命令

    # 看到下面的提示符代表成功,以后运行 Jumpserver 都要先运行以上 source 命令,载入环境后默认以下所有命令均在该虚拟环境中运行
    (py3) [root@localhost py3]

    # 下载 Jumpserver
    $ cd /opt/
    $ git clone https://github.com/jumpserver/jumpserver.git

    # 安装依赖 RPM 包
    $ yum -y install $(cat /opt/jumpserver/requirements/rpm_requirements.txt)

    # 安装 Python 库依赖
    $ pip install --upgrade pip setuptools
    $ pip install -r /opt/jumpserver/requirements/requirements.txt

.. code-block:: shell


    # 修改 Jumpserver 配置文件
    $ cd /opt/jumpserver
    $ cp config_example.py config.py
    $ vi config.py

**注意: 配置文件是 Python 格式,不要用 TAB,而要用空格**

.. code-block:: python

    """
        jumpserver.config
        ~~~~~~~~~~~~~~~~~

        Jumpserver project setting file

        :copyright: (c) 2014-2017 by Jumpserver Team
        :license: GPL v2, see LICENSE for more details.
    """
    import os

    BASE_DIR = os.path.dirname(os.path.abspath(__file__))


    class Config:
        """
        Jumpserver Config File
        Jumpserver 配置文件

        Jumpserver use this config for drive django framework running,
        You can set is value or set the same envirment value,
        Jumpserver look for config order: file => env => default

        Jumpserver使用配置来驱动Django框架的运行，
        你可以在该文件中设置，或者设置同样名称的环境变量,
        Jumpserver使用配置的顺序: 文件 => 环境变量 => 默认值
        """
        # SECURITY WARNING: keep the secret key used in production secret!
        # 加密秘钥 生产环境中请修改为随机字符串，请勿外泄
        SECRET_KEY = '2vym+ky!997d5kkcc64mnz06y1mmui3lut#(^wd=%s_qj$1%x'

        # SECURITY WARNING: keep the bootstrap token used in production secret!
        # 预共享Token coco和guacamole用来注册服务账号，不在使用原来的注册接受机制
        BOOTSTRAP_TOKEN = 'nwv4RdXpM82LtSvmV'

        # Development env open this, when error occur display the full process track, Production disable it
        # DEBUG 模式 开启DEBUG后遇到错误时可以看到更多日志
        # DEBUG = True
        DEBUG = False

        # DEBUG, INFO, WARNING, ERROR, CRITICAL can set. See https://docs.djangoproject.com/en/1.10/topics/logging/
        # 日志级别
        # LOG_LEVEL = 'DEBUG'
        # LOG_DIR = os.path.join(BASE_DIR, 'logs')
        LOG_LEVEL = 'ERROR'

        # Session expiration setting, Default 24 hour, Also set expired on on browser close
        # 浏览器Session过期时间，默认24小时, 也可以设置浏览器关闭则过期
        # SESSION_COOKIE_AGE = 3600 * 24
        # SESSION_EXPIRE_AT_BROWSER_CLOSE = False
        SESSION_EXPIRE_AT_BROWSER_CLOSE = True

        # Database setting, Support sqlite3, mysql, postgres ....
        # 数据库设置
        # See https://docs.djangoproject.com/en/1.10/ref/settings/#databases

        # SQLite setting:
        # 使用单文件sqlite数据库
        # DB_ENGINE = 'sqlite3'
        # DB_NAME = os.path.join(BASE_DIR, 'data', 'db.sqlite3')

        # MySQL or postgres setting like:
        # 使用Mysql作为数据库
        DB_ENGINE = 'mysql'
        DB_HOST = '127.0.0.1'
        DB_PORT = 3306
        DB_USER = 'jumpserver'
        DB_PASSWORD = 'weakPassword'
        DB_NAME = 'jumpserver'

        # When Django start it will bind this host and port
        # ./manage.py runserver 127.0.0.1:8080
        # 运行时绑定端口
        HTTP_BIND_HOST = '0.0.0.0'
        HTTP_LISTEN_PORT = 8080

        # Use Redis as broker for celery and web socket
        # Redis配置
        REDIS_HOST = '127.0.0.1'
        REDIS_PORT = 6379
        # REDIS_PASSWORD = ''
        # REDIS_DB_CELERY_BROKER = 3
        # REDIS_DB_CACHE = 4

        # Use OpenID authorization
        # 使用OpenID 来进行认证设置
        # BASE_SITE_URL = 'http://localhost:8080'
        # AUTH_OPENID = False  # True or False
        # AUTH_OPENID_SERVER_URL = 'https://openid-auth-server.com/'
        # AUTH_OPENID_REALM_NAME = 'realm-name'
        # AUTH_OPENID_CLIENT_ID = 'client-id'
        # AUTH_OPENID_CLIENT_SECRET = 'client-secret'

        def __init__(self):
            pass

        def __getattr__(self, item):
            return None


    class DevelopmentConfig(Config):
        pass


    class TestConfig(Config):
        pass


    class ProductionConfig(Config):
        pass


    # Default using Config settings, you can write if/else for different env
    config = DevelopmentConfig()

.. code-block:: shell

    # 生成数据库表结构和初始化数据
    $ cd /opt/jumpserver/utils
    $ sh make_migrations.sh

    # 运行 Jumpserver
    $ cd /opt/jumpserver
    $ ./jms start all  # 后台运行使用 -d 参数./jms start all -d
    # 新版本更新了运行脚本,使用方式./jms start|stop|status|restart all  后台运行请添加 -d 参数

.. code-block:: shell

    # 安装 docker 部署 coco 与 guacamole
    $ yum install -y yum-utils device-mapper-persistent-data lvm2
    $ yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
    $ yum makecache fast
    $ yum -y install docker-ce
    $ systemctl enable docker
    $ systemctl start docker

    # 注意,<Jumpserver_url> 请自行修改成 jumpserver 对外的访问地址,如 192.168.100.100:8080
    $ docker run --name jms_coco -d -p 2222:2222 -p 5000:5000 -e CORE_HOST=http://<Jumpserver_url> -e BOOTSTRAP_TOKEN=nwv4RdXpM82LtSvmV wojiushixiaobai/coco:1.4.5
    $ docker run --name jms_guacamole -d -p 8081:8081 -e JUMPSERVER_SERVER=http://<Jumpserver_url> -e BOOTSTRAP_TOKEN=nwv4RdXpM82LtSvmV wojiushixiaobai/guacamole:1.4.5

    # 允许 容器ip 访问宿主 8080 端口,(容器的 ip 可以进入容器查看)
    $ firewall-cmd --permanent --add-rich-rule="rule family="ipv4" source address="172.17.0.2" port protocol="tcp" port="8080" accept"
    $ firewall-cmd --permanent --add-rich-rule="rule family="ipv4" source address="172.17.0.3" port protocol="tcp" port="8080" accept"

    # 172.17.0.x 是docker容器默认的IP池

.. code-block:: shell

    # 安装 Web Terminal 前端: Luna  需要 Nginx 来运行访问 访问(https://github.com/jumpserver/luna/releases)下载对应版本的 release 包,直接解压,不需要编译
    $ cd /opt
    $ wget https://github.com/jumpserver/luna/releases/download/1.4.5/luna.tar.gz
    $ tar xf luna.tar.gz
    $ chown -R root:root luna

.. code-block:: shell

    # 配置 Nginx 整合各组件
    $ rm /etc/nginx/conf.d/default.conf

.. code-block:: shell

    $ vi /etc/nginx/conf.d/jumpserver.conf

    server {
        listen 80;

        client_max_body_size 100m;  # 录像及文件上传大小限制

        location /luna/ {
            try_files $uri / /index.html;
            alias /opt/luna/;  # luna 路径,如果修改安装目录,此处需要修改
        }

        location /media/ {
            add_header Content-Encoding gzip;
            root /opt/jumpserver/data/;  # 录像位置,如果修改安装目录,此处需要修改
        }

        location /static/ {
            root /opt/jumpserver/data/;  # 静态资源,如果修改安装目录,此处需要修改
        }

        location /socket.io/ {
            proxy_pass       http://localhost:5000/socket.io/;
            proxy_buffering off;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            access_log off;
        }

        location /coco/ {
            proxy_pass       http://localhost:5000/coco/;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            access_log off;
        }

        location /guacamole/ {
            proxy_pass       http://localhost:8081/;
            proxy_buffering off;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $http_connection;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            access_log off;
        }

        location / {
            proxy_pass http://localhost:8080;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }


.. code-block:: shell

    # 运行 Nginx
    $ nginx -t   # 确保配置没有问题, 有问题请先解决
    $ systemctl start nginx

    # 访问 http://192.168.244.144 (注意,没有 :8080,通过 nginx 代理端口进行访问)
    # 默认账号: admin 密码: admin  到会话管理-终端管理 接受 Coco Guacamole 等应用的注册
    # 测试连接
    $ ssh -p2222 admin@192.168.244.144
    $ sftp -P2222 admin@192.168.244.144
      密码: admin

    # 如果是用在 Windows 下,Xshell Terminal 登录语法如下
    $ ssh admin@192.168.244.144 2222
    $ sftp admin@192.168.244.144 2222
      密码: admin
      如果能登陆代表部署成功

    # sftp默认上传的位置在资产的 /tmp 目录下
    # windows拖拽上传的位置在资产的 Guacamole RDP上的 G 目录下

多组件负载说明

.. code-block:: shell

    # coco 服务默认运行在单核心下面, 当负载过高时会导致用户访问变慢, 这时可运行多个 docker 容器缓解
    $ docker run --name jms_coco01 -d -p 2223:2222 -p 5001:5000 -e CORE_HOST=http://<Jumpserver_url> -e BOOTSTRAP_TOKEN=nwv4RdXpM82LtSvmV wojiushixiaobai/coco:1.4.5
    $ docker run --name jms_coco02 -d -p 2224:2222 -p 5002:5000 -e CORE_HOST=http://<Jumpserver_url> -e BOOTSTRAP_TOKEN=nwv4RdXpM82LtSvmV wojiushixiaobai/coco:1.4.5
    ...

    # guacamole 也是一样
    $ docker run --name jms_guacamole01 -d -p 8082:8081 -e JUMPSERVER_SERVER=http://<Jumpserver_url> -e BOOTSTRAP_TOKEN=nwv4RdXpM82LtSvmV wojiushixiaobai/guacamole:1.4.5
    $ docker run --name jms_guacamole02 -d -p 8083:8081 -e JUMPSERVER_SERVER=http://<Jumpserver_url> -e BOOTSTRAP_TOKEN=nwv4RdXpM82LtSvmV wojiushixiaobai/guacamole:1.4.5
    ...

    # 注意开放防火墙, ip 请根据实际情况修改
    $ firewall-cmd --permanent --add-rich-rule="rule family="ipv4" source address="172.17.0.4" port protocol="tcp" port="8080" accept"
    $ firewall-cmd --permanent --add-rich-rule="rule family="ipv4" source address="172.17.0.5" port protocol="tcp" port="8080" accept"
    $ firewall-cmd --permanent --add-rich-rule="rule family="ipv4" source address="172.17.0.6" port protocol="tcp" port="8080" accept"
    $ firewall-cmd --permanent --add-rich-rule="rule family="ipv4" source address="172.17.0.7" port protocol="tcp" port="8080" accept"
    ...

    $firewall-cmd --reload

    # nginx 代理设置
    $ vi /etc/nginx.conf
    user  nginx;
    worker_processes  auto;

    error_log  /var/log/nginx/error.log warn;
    pid        /var/run/nginx.pid;


    events {
        worker_connections  1024;
    }

    # 加入 tcp 代理
    stream {
        log_format  proxy  '$remote_addr [$time_local] '
                           '$protocol $status $bytes_sent $bytes_received '
                           '$session_time "$upstream_addr" '
                           '"$upstream_bytes_sent" "$upstream_bytes_received" "$upstream_connect_time"';

        access_log /var/log/nginx/tcp-access.log  proxy;
        open_log_file_cache off;

        upstream cocossh {
            server localhost:2222 weight=1;
            server localhost:2223 weight=1;  # 多节点
            server localhost:2224 weight=1;  # 多节点
            # 这里是 coco ssh 的后端ip
            hash $remote_addr;
        }
        server {
            listen 2220;  # 不能使用已经使用的端口, 自行修改, 用户ssh登录时的端口
            proxy_pass cocossh;
            proxy_connect_timeout 10s;
            proxy_timeout 24h;   #代理超时
        }
    }
    # 到此结束

    http {
        include       /etc/nginx/mime.types;
        default_type  application/octet-stream;

        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for"';

        access_log  /var/log/nginx/access.log  main;

        sendfile        on;
        # tcp_nopush     on;

        keepalive_timeout  65;

        # 关闭版本显示
        server_tokens off;

        include /etc/nginx/conf.d/*.conf;
    }

    $ firewall-cmd --zone=public --add-port=2220/tcp --permanent
    $ firewall-cmd --reload

    $ vi /etc/nginx/conf.d/jumpserver.conf
    upstream jumpserver {
        server localhost:80;
        # 这里是 jumpserver 的后端ip
    }

    upstream cocows {
        server localhost:5000 weight=1;
        server localhost:5001 weight=1;  # 多节点
        server localhost:5002 weight=1;  # 多节点
        # 这里是 coco ws 的后端ip
        ip_hash;
    }

    upstream guacamole {
        server localhost:8081 weight=1;
        server localhost:8082 weight=1;  # 多节点
        server localhost:8083 weight=1;  # 多节点
        # 这里是 guacamole 的后端ip
        ip_hash;
    }

    server {
        listen 80;
        server_name demo.jumpserver.org;  # 自行修改成你的域名

        client_max_body_size 100m;  # 录像上传大小限制

        location / {
            proxy_pass http://jumpserver;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            access_log off;
        }

        location /luna/ {
            try_files $uri / /index.html;
            alias /opt/luna/;
        }

        location /socket.io/ {
            proxy_pass       http://cocows/socket.io/;  # coco
            proxy_buffering off;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            access_log off;
        }

        location /coco/ {
            proxy_pass       http://cocows/coco/;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            access_log off;
        }

        location /guacamole/ {
            proxy_pass       http://guacamole/;  #  guacamole
            proxy_buffering off;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $http_connection;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            access_log off;
        }
    }

    $ nginx -t
    $ nginx -s reload

后续的使用请参考 `快速入门 <admin_create_asset.html>`_
如遇到问题可参考 `FAQ <faq.html>`_
