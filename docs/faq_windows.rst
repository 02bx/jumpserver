Windows 资产连接说明
----------------------------

Windows 资产连接错误排查思路

::

    (1). 如果白屏  检查nginx配置文件的guacamole设置ip是否正确，检查终端管理的gua状态是否在线，检查资产设置及系统用户是否正确；
    (2). 如果显示没有权限 可能是你在 终端管理里没有接受 guacamole的注册，请接受一下
    (3). 如果显示未知问题 可能是你的资产填写的端口不对，或者授权的系统用户的协议不是rdp
    (4). 提示无法连接服务器，请联系管理员或查看日志 一般情况下是登录的系统账户不正确或者防火墙设置有误，可以从Windows的日志查看信息（资产的信息填写不正确也会报这个错误）
    (5). 提示网络问题无法连接或者超时，请检查网络连接并重试，或联系管理员 一般情况下网络有问题，可以从Windows的日志查看信息（资产的信息填写不正确也会报这个错误）

1. 检查终端是否在线

::

    注：连接 Windows 资产提示连接错误，您没有权限访问此连接，请按照此步骤解决
    # 如果终端不在线，请检查 Windows 组件是否已经正常运行，可用 docker ps 命令查询，安装文档有说明

    # 重新注册 Windows 组件
    # 在Jumpserver后台 会话管理-终端管理 删掉 guacamole 的注册
    $ docker stop jms_guacamole  # 如果名称更改过或者不对，请使用docker ps 查询容器的 CONTAINER ID ，然后docker stop <CONTAINER ID>
    $ docker rm jms_guacamole  # 如果名称更改过或者不对，请使用docker ps -a 查询容器的 CONTAINER ID ，然后docker rm <CONTAINER ID>
    $ rm /opt/guacamole/key/*  # guacamole, 如果你是按文档安装的，key应该在这里，如果不存在直接下一步
    $ systemctl stop docker
    $ systemctl start docker
    $ docker run --name jms_guacamole -d \
      -p 8081:8080 -v /opt/guacamole/key:/config/guacamole/key \
      -e JUMPSERVER_KEY_DIR=/config/guacamole/key \
      -e JUMPSERVER_SERVER=http://<填写jumpserver的url地址> \
      jumpserver/guacamole:latest

    # 如果镜像不是jumpserver/guacamole 请更换 registry.jumpserver.org/public/guacamole

    # 正常运行后到Jumpserver 会话管理-终端管理 里面接受gua注册
    $ docker restart jms_guacamole  # 如果接受注册后显示不在线，重启gua就好了

.. image:: _static/img/faq_windows_01.jpg

2. 登录要连接的windows资产，检查远程设置和防火墙设置

::

    # Windows 7/2008 勾选 允许运行任意版本远程桌面的计算机连接（较不安全）（L）
    # Windows 8/10/2012 勾选 允许远程连接到此计算机（L）

    # Windows防火墙-高级设置-入站规则 把远程桌面开头的选项 右键-启用规则
    # Windows 7/2008 启用 远程桌面(TCP-In)
    # Windows 8/10/2012 启用 远程桌面-用户模式(TCP-In) 远程桌面-用户模式(UDP-In)

3. 登录要连接的windows资产，检查用户和IP信息（Windows目前还不支持推送，所以必须使用资产上面已存在的用户进行登录）

::

    # 注：因为 windows 暂时不支持推送，所以必须使用资产上面已经存在的账户进行登录，如 administrator 账户

.. image:: _static/img/faq_windows_02.jpg

4. 创建Windows资产管理用户（如果是域资产，格式是uesr@domain.com）

::

    # 不带域的用户直接输入用户名即可，如 administrator
    # 域用户的用户名格式为 user@domain.com，如 administrator@jumpserver.org

.. image:: _static/img/faq_windows_03.jpg

5. 创建Windows资产系统用户（如果是域资产，格式是uesr@domain.com，注意协议不要选错）

::

    # 注：因为 windows 暂时不支持推送，所以必须使用资产上面已经存在的账户进行登录，如 administrator 账户
    # 不带域的用户直接输入用户名即可，如 administrator
    # 域用户的用户名格式为 user@domain.com，如 administrator@jumpserver.org
    # 如果想让用户登录资产时自己输入资产的账户密码，可以点击系统用户的名称 点击清除认证信息
    # 此处必须输入能正确登录 windows 资产的 账户密码
    # 如不确实是不是因为密码或者账户信息错误导致的无法登录，可以使用清除认证信息或者手动登录功能（在系统用户处设置）

.. image:: _static/img/faq_windows_04.jpg

6. 创建Windows资产（注意端口不要填错）

.. image:: _static/img/faq_windows_05.jpg

7. 创建授权规则

::

    # 先定位到 windows 的资产，然后授权，如果资产用户密码不一致，请不要直接在节点上授权

.. image:: _static/img/faq_windows_06.jpg

8. 使用web terminal登录（如果登录报错，检查防火墙的设置，可以参考FAQ）

.. image:: _static/img/faq_windows_07.jpg

9. 上传文件到 windows

::

    # 直接拖拽文件到 windows 窗口即可，文件上传后在 Guacamole RDP上的 G 目录查看

.. image:: _static/img/faq_windows_08.jpg

其他问题可参考 `FAQ <faq.html>`_
