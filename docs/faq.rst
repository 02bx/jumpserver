FAQ
==========
.. toctree::
   :maxdepth: 1

   LDAP 使用说明 <faq_ldap.rst>
   SFTP 使用说明 <faq_sftp.rst>
   TELNET 使用说明 <faq_telnet.rst>
   Docker 使用说明 <faq_docker.rst>
   安装过程 常见问题 <faq_install.rst>
   Firewalld 使用说明 <faq_firewalld.rst>
   RDP 协议资产连接说明 <faq_rdp.rst>
   SSH 协议资产连接说明 <faq_ssh.rst>
   添加组织 及 组织管理员说明 <faq_org.rst>
   二次认证(Google Auth)入口说明 <faq_googleauth.rst>


其他问题
~~~~~~~~~~~~~~~~~~~~~

1. 用户、系统用户、管理用户的关系

.. code-block:: vim

    # 用户管理里面的用户列表 是用来登录jumpserver平台的用户,用户需要先登录jumpserver平台,才能管理或者连接资产
    # 资产管理里面的管理用户 是jumpserver用来管理资产需要的服务账户,Linux资产需要root或 NOPASSWD: ALL sudo权限,Jumpserver使用该用户来 '推送系统用户'、'获取资产硬件信息'等。Windows资产随意指定一个,暂无作用
    # 资产管理里面的系统用户 是jumpserver用户连接资产需要的登录账户,Linux资产可以自动推送该系统用户到资产上,Windows需要指定资产上已经创建的系统用户

2. input/output error, 通常jumpserver所在服务器字符集问题

.. code-block:: shell

    # Centos7
    $ localedef -c -f UTF-8 -i zh_CN zh_CN.UTF-8
    $ export LC_ALL=zh_CN.UTF-8
    $ echo 'LANG="zh_CN.UTF-8"' > /etc/locale.conf

    # Centos6
    $ localedef -c -f UTF-8 -i zh_CN zh_CN.UTF-8
    $ export LC_ALL=zh_CN.UTF-8
    $ echo 'LANG="zh_CN.UTF-8"' > /etc/sysconfig/i18n

    # Ubuntu
    $ apt-get install language-pack-zh-hans
    $ echo 'LANG="zh_CN.UTF-8"' > /etc/default/locale

    如果任然报input/output error,尝试执行 yum update 后重启服务器(仅测试中参考使用,实际运营服务器请谨慎操作)

3. luna 无法访问

.. code-block:: vim

    # Luna 打开网页提示403 Forbidden错误,一般是nginx配置文件的luna路径不正确或者luna下载了源代码,请重新下载编译好的代码
    # Luna 打开网页提示502 Bad Gateway错误,一般是selinux和防火墙的问题,请根据nginx的errorlog来检查

4. 录像问题

.. code-block:: vim

    # 默认录像存储位置在jumpserver/data/media  可以通过映射或者软连接方式来使用其他目录

    # 录像和命令记录存储到其他位置,可以到 Jumpserver 系统设置-终端设置 里面进行设置

    # 修改后,需要修改在Jumpserver 会话管理-终端管理 修改terminal的配置 录像存储 命令记录
    # 注意,命令记录需要所有保存地址都正常可用,否则 历史会话 和 命令记录 页面无法正常访问

5. 在终端修改管理员密码及新建超级用户

.. code-block:: shell

    # 管理密码忘记了或者重置管理员密码
    $ source /opt/py3/bin/activate
    $ cd /opt/jumpserver/apps
    $ python manage.py changepassword  <user_name>

    # 新建超级用户的命令如下命令
    $ python manage.py createsuperuser --username=user --email=user@domain.com

6. 修改登录超时时间(默认 10 秒)

.. code-block:: shell

    $ vim /opt/coco/conf.py

    # 把 SSH_TIMEOUT = 15 修改成你想要的数字 单位为：秒
    SSH_TIMEOUT = 60

7. 升级提示 Table 'xxx' already exists

.. code-block:: shell

    # 数据库表结构文件丢失,请参考离线升级文档,把备份的数据库表文件还原

    # 可以使用如下命令检查丢失了什么表结构
    $ cd /opt/jumpserver/apps
    $ for d in $(ls); do if [ -d $d ] && [ -d $d/migrations ]; then ll ${d}/migrations/*.py | grep -v __init__.py; fi; done

    # 新开一个终端
    $ mysql -uroot -p
    > use jumpserver;
    > select app,name from django_migrations where app in('assets','audits','common','ops','orgs','perms','terminal','users') order by app asc;

    # 对比即可知道丢失什么文件,把文件从备份目录拷贝即可

8. 设置浏览器过期

.. code-block:: shell

    $ vim /opt/jumpserver/apps/jumpserver/settings.py

.. code-block:: python

    # 找到如下行,注释(可参考 django 设置 session 过期时间),修改你要的设置即可
    # SESSION_COOKIE_AGE = CONFIG.SESSION_COOKIE_AGE or 3600 * 24

    # 如下,设置关闭浏览器 cookie 失效,则修改为
    # SESSION_COOKIE_AGE = CONFIG.SESSION_COOKIE_AGE or 3600 * 24
    SESSION_EXPIRE_AT_BROWSER_CLOSE = True

9. MFA遗失无法登陆

.. code-block:: vim

    普通用户联系管理员关闭MFA,登录成功后用户在个人信息里面重新绑定.
    如果管理员遗失无法登陆, 修改数据库 users_user 表对应用户的 otp_level 为 0 , 重新登陆绑定即可
    如果在系统设置里面开启的 MFA 二次认证 ,需要修改数据库 settings 表 SECURITY_MFA_AUTH 的 value 值为 false

10. 资产授权说明

.. code-block:: vim

    资产授权就是把 系统用户关联到用户 并授权到 对应的资产
    用户只能看到自己被授权的资产

11. Web Terminal 页面经常需要重新刷新页面才能连接资产

.. code-block:: nginx

    # 具体表现为在luna页面一会可以连接资产,一会就不行,需要多次刷新页面
    # 如果从开发者工具里面看,可以看到部分不正常的 502 socket.io
    # 此问题一般是由最前端一层的nginx反向代理造成的,需要在每层的代理上添加(注意是每层)
    $ vim /etc/nginx/conf.d/jumpserver.conf  # 配置文件所在目录,自行修改

    ...  # 省略

    location /socket.io/ {
            proxy_pass http://你后端的服务器url地址/socket.io/;
            proxy_buffering off;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            access_log off;  # 不记录到 log
    }

    location /guacamole/ {
            proxy_pass       http://你后端的服务器url地址/guacamole/;
            proxy_buffering off;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $http_connection;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            access_log off;  # 不记录到 log
    }
    ...

    # 为了便于理解,附上一份 demo 网站的配置文件参考
    $ vim /etc/nginx/conf.d/jumpserver.conf
    server {

        listen 80;
        server_name demo.jumpserver.org;

        client_max_body_size 100m;  # 上传录像大小限制

        location / {
                # 这里的IP是后端服务器的IP,后端服务器就是文档一步一步安装来的
                proxy_pass http://192.168.244.144;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header Host $host;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_read_timeout 150;
        }

        location /socket.io/ {
                proxy_pass http://192.168.244.144/socket.io/;
                proxy_buffering off;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "upgrade";
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header Host $host;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        location /guacamole/ {
                proxy_pass       http://192.168.244.144/guacamole/;
                proxy_buffering off;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection $http_connection;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header Host $host;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }

12. 连接资产时提示 System user <xxx> and asset <xxx> protocol are inconsistent.

.. code-block:: shell

    # 这是因为系统用户的协议和资产的协议不一致导致的
    # 检查系统用户的协议和资产的协议
    # 如果是更新了版本 Windows资产 出现的问题,请执行下面代码解决
    $ source /opt/py3/bin/activate
    $ cd /opt/jumpserver/utils
    $ sh 2018_07_15_set_win_protocol_to_ssh.sh

    # 如果不存在 2018_07_15_set_win_protocol_to_ssh.sh 脚本,可以手动执行下面命令解决
    $ source /opt/py3/bin/activate
    $ cd /opt/jumpserver/apps
    $ python manage.py shell
    >>> from assets.models import Asset
    >>> Asset.objects.filter(platform__startswith='Win').update(protocol='rdp')
    >>> exit()


13. 重启服务器后无法访问 Jumpserver,页面提示502 或者 403等

.. code-block:: shell

    # CentOS 7 临时关闭
    $ setenforce 0  # 临时关闭 selinux,重启后失效
    $ systemctl stop firewalld.service  # 临时关闭防火墙,重启后失效

    # Centos 7 如需永久关闭,还需执行下面步骤
    $ sed -i "s/enforcing/disabled/g" 'grep enforcing -rl /etc/selinux/config'  # 禁用 selinux
    $ systemctl disable firewalld.service  # 禁用防火墙

    # Centos 7 在不关闭 selinux 和 防火墙 的情况下使用 Jumpserver
    $ firewall-cmd --zone=public --add-port=80/tcp --permanent  # nginx 端口
    $ firewall-cmd --zone=public --add-port=2222/tcp --permanent  # 用户SSH登录端口 coco
      --permanent  永久生效,没有此参数重启后失效

    $ firewall-cmd --reload  # 重新载入规则

    $ setsebool -P httpd_can_network_connect 1  # 设置 selinux 允许 http 访问
    $ chcon -Rt svirt_sandbox_file_t /opt/guacamole/key  # 设置 selinux 允许容器对目录读写

14. 生成随机 SECRET_KEY

.. code-block:: shell

    $ source /opt/py3/bin/activate
    $ cd /opt/jumpserver/apps
    $ python manage.py shell
    >>> from django.core.management.utils import get_random_secret_key
    >>> get_random_secret_key()

15. 传递明文数据到 Jumpserver 数据库(数据导入)

.. code-block:: shell

    # 以导入 admin 用户 public_key 为例
    $ source /opt/py3/bin/activate
    $ cd /opt/jumpserver/apps
    $ python manage.py shell
    >>> from users.models import User
    >>> user = User.objects.get(username='admin')
    >>> user.public_key = '明文key'
    >>> user.save()
