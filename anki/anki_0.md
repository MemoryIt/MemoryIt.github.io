# 在aliyun搭建anki服务器，在多端实现anki同步
2021-12-10

2021-12-15 不要使用anki2.1调度算法
## 通过docker安装anki-sync-server[地址](https://github.com/ankicommunity/anki-sync-server)
1. 从docker-hub下载anki-sync-server的docker镜像[地址](https://hub.docker.com/r/kuklinistvan/anki-sync-server)
    ```shell
    #从docker-hub拉取（下载）镜像到本地。
    docker pull kuklinistvan/anki-sync-server:latest
    #列出本机所有docker镜像，查看目标镜像是否下载成功。
    docker image ls
    ```
2. 在宿主机创建用于存储anki相关文件的目录
    ```shell
    #进入/home目录
    cd /home
    
    #为anki数据和脚本新建一个名为anki的目录
    mkdir anki
    
    #进入/home/anki目录，在此目录下创建用于存储同步数据的anki-data目录
    cd /home/anki
    mkdir anki-data
    ```
3. 在/home/anki目录下新建脚本run.sh，用于在docker中启动anki-server服务。(copy from [刘阳](https://handsomeliuyang.github.io/2020/01/08/%E7%BB%8F%E9%AA%8C%E6%80%BB%E7%BB%93/%E4%BA%91%E6%9C%8D%E5%8A%A1%E6%90%AD%E5%BB%BAAnki-Sync-Server/index.html))
    ```shell
    export DOCKER_USER=root 
    export ANKI_SYNC_DATA_DIR=/home/anki/anki-data
    #anki-server容器用于和外界通信的宿主机端口。
    export HOST_PORT=25501

    #更改anki-data文件夹属性
    #所有者：root用户
    chown "$DOCKER_USER" "$ANKI_SYNC_DATA_DIR"
    #属性：root用户-rwx(可读可写可运行) 其他用户(读写运行都不可)
    chmod 700 "$ANKI_SYNC_DATA_DIR"

    #第一行：-i让容器的标准输入保持打开、-t让docker分配一个伪终端并绑定到容器的标准输入上、-d让docker在后台运行，不把结果输出到当前宿主机下。
    #第二行：在宿主机和anki容器之间建立文件夹映射(从source映射到target)
    #第三行：在宿主机和anki容器之间建立网络映射(从25501端口映射到27701)
    #第四行：指定本容器名称为 anki-container。
    #第五行：docker重启时，本容器自动启动。
    #第六行：指定使用哪一个镜像。
    docker run -itd \
        --mount type=bind,source="$ANKI_SYNC_DATA_DIR",target=/app/data \
        -p "$HOST_PORT":27701 \
        --name anki-container \
        --restart always \
        kuklinistvan/anki-sync-server:latest
    ```
4. 运行docker容器并添加anki用户信息
   ```shell
   #启动docker容器
   bash /home/anki/run.sh
   
   #进入docker容器
   docker exec -it anki-container /bin/sh
   #运行这个命令之后，会进入到容器内/app/anki-sync-server文件夹
   
   #查看添加用户的命令帮助
   ./ankisyncctl.py --help
   
   #添加新用户并设置密码
   ./ankisyncctl.py adduser <username>
   
   #列出所有的用户
   ./ankisyncctl.py lsuser
   ```
5. 设置aliyun安全组规则，只允许特定ip(自己家)通过tcp协议访问服务器的25501端口。

    默认情况下大部分端口入方向都是关闭的，需要手动添加入方向的访问规则，协议类型：tcp，端口范围：25501/25501，授权对象：家里的ip地址。


## 各平台anki客户端同步测试
为了避免各种奇奇怪怪bug的出现，直接使用anki-community在各平台测试过的指定版本的客户端。[地址](https://github.com/ankicommunity/anki-devops-services#tested-client-downloads)

1. android anki 2.9.1
    
    【设置】-->【高级设置】-->【自定义同步服务器】-->更改同步地址和媒体文件同步地址中的ip和端口号为自己的设置(注意使用的是http协议)-->【设置】-->【ankidroid】-->【ankiweb账户】-->输入自己在服务器上设置的用户名和密码进行登陆。-->返回主页面，点击右上角的圈圈进行同步。
2. ubuntu 20.04(Kernel: 5.11.0-41) anki 2.1.19

    安装SyncRedirector插件，更改同步服务器ip，使用之前设置的用户信息登陆，显示同步失败。
    ```shell
    同步失败: 服务器没有找到。或者你的链接是中断的，或者杀毒软件/防火墙软件正在阻止Anki链接Internet。
    #然而我并没有开启ufw,放弃使用该插件。
    ```
    使用anki-community给出的解决方案,在anki插件目录下新建一个目录(比如说叫ankisyncd),然后在该目录下新建一个名为`__init__.py`的文件(注意:init前后各有两个下划线),将以下内容复制到该文件中(注意:把ip和端口修改为自己的)。
    ```shell
    import anki.sync, anki.hooks, aqt

    #注意:使用http协议，https不能进行同步。
    #将127.0.0.1修改为自己服务器的公网ip。
    #27701是anki-server的默认服务端口，anki-server运行在docker容器内，之前将服务器本地端口25501映射到了anki-server容器的27701端口。所以这里要把默认端口号改为25501
    addr = "http://127.0.0.1:27701/"
    
    anki.sync.SYNC_BASE = "%s" + addr
    def resetHostNum():
        aqt.mw.pm.profile['hostNum'] = None
    anki.hooks.addHook("profileLoaded", resetHostNum)
    ```
3. windows 10 anki 2.1.19
    
    解决方法同ubuntu。
    

## 同步测试

+ 使用ankidroid和桌面端修改牌组配置后，发现桌面端的anki同步失败，ankidroid仍可以进行同步。发现2.1.49版本更新中有[提到](https://github.com/ankitects/anki/releases/tag/2.1.49)：`Work around an AnkiDroid inconsistency causing deck config to be reset if options edited on AnkiDroid.`所以怀疑是在移动端编辑了牌组选项导致了这个同步失败。在anki-server中新建test用户用于测试。
+ 测试1：不修改牌组默认选项，仅使用桌面端(win&ubuntu)
    + 在win端导入牌组，同步至服务器，在ubuntu端下载牌组。
    + 在ubuntu端对牌组进行学习，同步学习进度到服务器，在win端从服务器同步新的学习进度，再学习几张牌组，之后同步进度到服务器。
    + 在ubuntu端导入牌组，同步至服务器，再同步到win端。
+ 测试2：修改牌组默认选项，仅使用桌面端(win&ubuntu)
    + win端修改，同步到ubuntu端。
    + ubuntu端修改，同步到win端。
+ 测试3：使用anki 2.1 调度算法
    + 同步失败，看来这个docker镜像不能使用anki2.1调度算法。
+ 测试4：仅使用2.0调度算法，在三端同步。
    + 同步成功。
+ 测试5：使用2.0调度算法，分别更改牌组选项设置，在三端同步。
    + 同步成功。


结论：anki2.1调度算法导致了这个问题，桌面端不要使用 [Anki 2.1 scheduler算法](https://faqs.ankiweb.net/the-anki-2.1-scheduler.html)，android端不要勾选实验性V2调度器。

## 参考
1. https://github.com/ankicommunity/anki-sync-server
2. https://hub.docker.com/r/kuklinistvan/anki-sync-server
3. https://github.com/ankicommunity/anki-devops-services#tested-client-downloads
4. https://handsomeliuyang.github.io/2020/01/08/%E7%BB%8F%E9%AA%8C%E6%80%BB%E7%BB%93/%E4%BA%91%E6%9C%8D%E5%8A%A1%E6%90%AD%E5%BB%BAAnki-Sync-Server/index.html
