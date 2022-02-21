# 威纶通NAS笔记

## 一、存储与快照总管

### 1、存储

空的磁盘上可以创建：存储池、静态卷；

存储池：可以是一个磁盘，也可以是几个磁盘（HDD x n，或者（SSD x n + HDD x m））组成的raid阵列上形成的空间。它无法直接存放数据，还需在上面新建卷或者LUN，方可读写数据；

静态卷：它也可以是一个磁盘或者几个磁盘共同组成的一个大磁盘空间。它可以直接存储数据；

存储池上可以创建：厚卷、精简卷（薄卷）、区块LUN（相当于Windows硬盘驱动器，如D盘）；

厚卷、精简卷、静态卷是磁盘阵列上的概念，而[LUN](https://www.cnblogs.com/dion-90/articles/8577254.html)（LogicUnitNumber）分为区块LUN和文件LUN，是SCSI上的概念，它是在SCSI-3中定义的，而并非单用于存储范畴，也可以指使用SCSI协议的一切外围设备，如磁带机、SCSI打印机等等。；

> 从SCSI-3的SAM模型中我们知道，SCSI-3（或者之后的版本）的协议层规定，对于16位宽的SCSI总线，其寻址范围只有16个，即只能挂载16个外围设备，每个设备称为一个target。为了提高总线的寻址能力，于是又引入了一层，它规定在每个target上，还可以虚拟（也可以实际连接）出多个设备，例如某个target上可能接了一个磁带机，一个打印机，他们共用一个target地址，但为了区分他们，于是就用LUN加以区别，磁带机假设为LUN0，打印机假设为LUN2，这样就解决了多设备的寻址问题。
>
> 　　这是实际设备连接的例子，存储阵列（比如：HP leftHand P4000 SAN）是最好的虚拟设备的例子。一个存储磁盘阵列在SCSI总线看来是一个Target，占用一个SCSI的Target地址，但存储阵列的存储空间太大，我们需要将其分成不同的部分，以供不同的应用，达到集中存储，集中管理的目的。所以在分割出来的每个存储部分（或区域）我们就用Lun来区别，如LUN1代表地址块0-1023，LUN2代表地址块1024-65535等等。从上面可以看出，计算机在使用SCSI标准（注意我这里用的标准一词，代表了统含SAM模型中的4层，而并不使用接口，协议或者命令等词语）接外挂存储时，使用的是总线（BUS）-目标（Target）-LUN三元寻址方案，总线指的是你的计算机上有几条SCSI总线，有几块SCSI卡？目标指的是在该总线上，设备的目标地址即常说的SCSI地址是多少？LUN指的是设备在一个Target上分配的逻辑地址，逻辑单元号。这种寻址方案和设备的连接方式，类似于物理上星形连接，逻辑上总线连接的一种网络拓扑。
>
> []: https://www.cnblogs.com/dion-90/articles/8577254.html	"关于LUN和存储卷的区别详解"

在威联通QTS系统中，厚卷最为常见，它的空间大小要求预先在存储池上分配好并占用，且数据存满之后将不能继续存放数据。但是它允许对其从存储池里获取空间进行扩大，或者对其缩小，释放存储池空间。扩大空间很容易，快速且不需要删除快照，缩小空间则相对较慢，且需要删除快照。

在威联通QTS系统中，精简卷（薄卷）则不像厚卷那样预先占用分配好的空间，它采用“订阅”空间的方式。订阅空间的大小可以超过目前现有存储池的大小，以待后期增加硬盘扩大存储池大小。**订阅的空间可以被快照或者其他卷占用**。精简卷扩大空间不用删除快照，缩小提示也不用删除快照，仅仅要求停止其上运行的文件程序。另外，其中的“回收磁盘空间”不知道是干啥的。

当精简卷里的文件增长超过其订阅空间大小时，将会自己从存储池里获取空间进行增长。使用时应当注意其空间增长导致存储池空间不足，进而导致系统不稳定、快照恢复等情况；

厚卷、精简卷可以相互转换，厚卷转为精简卷需要删除快照，反之不用（如果同时缩小空间，可能会需要删除快照）；

厚卷、精简卷不能执行卸载磁盘，而静态卷像U盘一样，可以被安全卸载；

### 2、快照

在执行**系统盘还原**时，

1. 先在威联通`AppCente`r手动停止各项应用程序；
1. `ssh`登陆`admin`账户，在初始化命令行菜单中选择`Reboot in Maintenance Mode`；
2. 重启进入系统之后检查并关闭其他非系统应用；
3. 执行快照还原；

【注意事项】：

​        第一次在没有作上述准备时直接还原系统盘，结果卡在20%进度，查看system event logs，发现是在中止应用服务时卡住，随即直接重启NAS，按上面3步走后重新执行还原，还原速度很快，几分钟就成功。

​        同理，重启NAS之前，最好先手动停止各类服务，避免直接重启可能出现的卡住；

### 3、TS-264C快照部署

1. `SSD`盘（`存储池2`）中的`A_System`（精简卷）部署【本地快照】，储存在`存储池2`（`SSD`）中。计划任务为每天早上7点30。本地快照可在用户进行错误的操作后还原系统；

2. `A_System`（精简卷）还部署了【远程快照】，存储在`存储池1`（`HDD1`）中的快照保险库（`V_A_System`）中。计划任务为每周天早上8点。远程快照可在`SSD`损坏后，快速从`HDD1`处恢复系统盘`A_System`；

   ![快照保险库](https://easyimage.kingchl.cn:9999/i/2022/02/08/image.png)

3. `HDD1`（`存储池1`）中的`B_DataDynamic`（厚卷）部署【本地快照】，储存在存储池1中。计划任务为每天早上7点30 ；

4. `B_DataDynamic`未配置【远程快照】；

5. 文件级别备份请参照第四章“同步与备份：`HBS 3`“，`A_System`拟备份`Public`共享文件夹，`B_DataDynamic`备份`DYNAMIC`共享文件夹。均备份至`HDD2`（`STAITC/BACKUP`）中。注意，快照无法使用`HBS 3`备份；

## 二、ISCSI

### 1、新建iSCSI目标

### 2、新建LUN

新建的LUN可以基于存储池（区块），也可以基于卷（文件）。LUN可以配置厚LUN，也可以配置为精简LUN，这里建议选择精简LUN。

假如我们配置精简LUN大小为500G，那么一开始它在存储池的占用并没有500G，而是会慢慢增长至500G。LUN可以在不删除的数据的前提下扩大容量，但是不能减小容量。

新建好LUN之后，将其挂入第1步新建好的iSCSI目标中。

LUN对于Windows来说，就相当于一块来自远程NAS的虚拟硬盘。

### 3、win10端发起iSCSI连接

参考如下链接：

[Windows10系统连接iscsi target（共享存储）-百度经验 (baidu.com)](https://jingyan.baidu.com/article/e4511cf37feade2b845eaff8.html)

### 4、新建磁盘

进入win10的磁盘管理，格式化得到的新虚拟磁盘并分区。

## 三、硬盘休眠

参考：

[一次QNAP威联通系统迁移（硬件TVS-951N，系统4.5.1.1495）](https://post.smzdm.com/p/ar0nnmkz/)

[在威联通NAS上完美实现硬盘单独休眠](http://www.nasyun.com/thread-67376-1-1.html)



## 四、同步与备份

### 1、Qsync Central

威联通Nas中，该服务提供客户端（Windows、Mac、Android、IOS）与nas的`File Station`数据实时同步功能。进而实现多客户端之间的数据实时同步；

**注意事项：**

建议采用禁用“节省空间模式”，同时禁用“智能删除”功能。然后配合seafile完成“笔记本→（广域网）→seafile→台式机→（局域网）→Qsync→NAS动态盘→HBS 3→NAS静态备份盘”

如果非要采用“节省空间模式”，建议对win10上的同步文件（/文件夹）采用“始终保留在此设备上”，或“在此设备上本地可用”的“节省空间模式”；

当“节省空间模式”采用“释放空间但保留在NAS上”时，win10上的文件（/文件夹）属性变为“联机时可用”，文件实质是存在NAS上，但是保留文件信息。当seafile同步此属性的文件时，Qsync的win10客户端不能退出或者断开与NAS的连接，否则当该文件发生更改时，seafile将找不到该文件的具体内容，因为它时存在于NAS上的。

【纯猜测】另外关于“始终在此设备上可用”与“在此设备上可用”这两种文件属性的猜测，当同步文件夹里的文件被配置为“始终在此设备上可用”时，该文件应该不会被win10的存储感知给空间优化掉，而后者则是能够被存储感知优化成“联机时可用”的文件属性。

当尝试删除掉整个同步根文件夹时，如果只有seafile客户端打开的情况下，该文件夹可以从win10删除，同时seafile断开与此同步根文件夹的连接，数据依然存在于seafile服务器里。在seafile里重新连接到本地该同步根文件夹时，会合并二者不同的数据，相同的数据应该是比较那个修改时间较新；

当尝试删除掉整个同步根文件夹时，如果Qsync客户端在线时，无法删除整个同步根文件夹（提示被占用），也无法删除该文件夹内的“非联机文件”（“始终在此设备上可用”与“在此设备上可用”这两种）。而“联机文件”由于不在win10本地，因而会被删除（相当于本地执行删除操作），同时seafile服务端、Qsync服务端也会同步此删除。

seafile的分享与协作能力比Qsync好用，而后者的日志比前者齐全。

### 2、HBS 3

1. 任务：Backup  of ARCHIVE，每周六早8点。版本管理：保留3个版本。路径：`/STATIC/BACKUP/'Backup of ARCHIVE.qdff'`；
2. 任务：Backup of Public，预留；
3. 这些`.qdff`文件后面都保存到百度网盘中去，单向同步；

## 五、VirtualizationStation 3

### 1、虚拟机工作站的普遍配置



### 2、虚拟机Windows 10 @TS-264C

#### a）映像来源

百度网盘，映像名：（Qnap虚拟机在用）Win10Ltsb2016x64.leon.v6.ESD；

#### b）注意事项

VirtualizationStation 3的配置中，“虚拟内存”项目中的“**启用动态内存**”复选框不要勾选！否则在同时开启其他虚拟机时，此精简版本的win10系统会出现哭脸蓝屏崩溃；

### 3、虚拟机Ubuntu LTS @TS-264C

#### a）映像来源

源自台式机中Ubuntu20.04LTS虚拟机，移植到此，用于seafile；

#### b）注意事项

后期可以通过备份.img文件

## 六、QTS系统管理

### 0、QTS登录管理

SSH端口号：2022（原缺省为：22）；

HTTPS（SSL）：https://kingchl.top:5001（阿帕奇）

HTTPS：https://kingchl.asuscomm.com:5001（阿帕奇）

HTTP内网：http://192.168.50.61:5000（阿帕奇）

管理员：`admin`，密码：**`245EBE58CE40`**

### 1、QTS系统构造

根目录组成及文件夹映射：

```bash
[/] # ll
total 4.0K
drwxr-xr-x   ./
drwxr-xr-x   ../
-rw-rw-rw-    1
drwxr-xr-x    bin/
drwxr-xr-x    dev/
drwxr-xr-x    etc/
drwxr-xr-x    flashfs_tmp/
lrwxrwxrwx    hd_root_tmp@ -> /mnt/HDA_ROOT/
drwxr-xr-x    home/
drwxr-xr-x    lib/
lrwxrwxrwx    lib64@ -> lib/
lrwxrwxrwx    linuxrc@ -> bin/busybox*
drwx------    lost+found/
drwxr-xr-x    mnt/
drwxr-xr-x    new_root/
lrwxrwxrwx    opt@ -> /share/CACHEDEV1_DATA/.qpkg/Entware/
lrwxrwxrwx    php.ini@ -> /etc/config/php.ini
dr-xr-xr-x    proc/
lrwxrwxrwx    QVS@ -> /share/CACHEDEV1_DATA/.qpkg/QKVM/
-rw-------    .rnd
drwxr-xr-x    root/
drwxr-xr-x    rpc/
drwxr-xr-x    run/
drwxrwxrwt    samba_third_party/
drwxr-xr-x    sbin/
drwxrwxrwt    share/
dr-xr-xr-x    sys/
drwxrwxrwx    tmp/
drwxr-xr-x    usr/
drwxr-xr-x    var/

```

`@`代表文件夹快捷方式

QTS应用软件安装路径：

```bash
/share/CACHEDEV1_DATA/.qpkg/
```

Entware应用软件映射路径：

```bash
/opt@ -> /share/CACHEDEV1_DATA/.qpkg/Entware/
```



### 2、`CONTAINER`

`QTS`系统支持`docker`、`lxc`、`lxd`三种容器技术，这里以`docker`为主；

#### A）Docker容器概述

一般`linux`系统中`docker`工作路径为`var/lib/docker`，`QTS`系统中，`docke`r的工作路径为：

```bash
# docker安装路径
/share/CACHEDEV1_DATA/Public/Container/container-station-data/lib/docker/
# docker volume路径
/share/CACHEDEV1_DATA/Public/Container/container-station-data/lib/docker/volumes/
# bind mount位置（默认持久化目录，可自定义）
/share/CACHEDEV1_DATA/Public/Container/Docker/
```

> Docker的数据持久化主要有两种方式：
>
> - bind mount
> - volume
>
> Docker的数据持久化即使数据不随着container的结束而结束，数据存在于host机器上——要么存在于host的某个指定目录中（使用bind mount），要么使用docker自己管理的volume（/var/lib/docker/volumes下）。
>
> ### bind mount
>
> bind mount自docker早期便开始为人们使用了，用于将host机器的目录mount到container中。但是bind mount在不同的宿主机系统时不可移植的，比如Windows和Linux的目录结构是不一样的，bind mount所指向的host目录也不能一样。这也是为什么bind mount不能出现在Dockerfile中的原因，因为这样Dockerfile就不可移植了。
>
> 将host机器上当前目录下的host-data目录mount到container中的/container-data目录：
>
> ```bash
> docker run -it -v $(pwd)/host-dava:/container-data alpine sh
> ```
>
> 有几点需要注意：
>
> - host机器的目录路径必须为全路径(准确的说需要以`/`或`~/`开始的路径)，不然docker会将其当做volume而不是volume处理
> - 如果host机器上的目录不存在，docker会自动创建该目录
> - 如果container中的目录不存在，docker会自动创建该目录
> - 如果container中的目录已经有内容，那么docker会使用host上的目录将其覆盖掉
>
> ### 使用volume
>
> volume也是绕过container的文件系统，直接将数据写到host机器上，只是volume是被docker管理的，docker下所有的volume都在host机器上的指定目录下/var/lib/docker/volumes。
>
> 将my-volume挂载到container中的/mydata目录：
>
> ```bash
> docker run -it -v my-volume:/mydata alpine sh
> ```
>
> 然后可以查看到给my-volume的volume：
>
> ```bash
> docker volume inspect my-volume
> [
>     {
>         "CreatedAt": "2018-03-28T14:52:49Z",
>         "Driver": "local",
>         "Labels": null,
>         "Mountpoint": "/var/lib/docker/volumes/my-volume/_data",
>         "Name": "my-volume",
>         "Options": {},
>         "Scope": "local"
>     }
> ]
> ```
>
> 可以看到，volume在host机器的目录为`/var/lib/docker/volumes/my-volume/_data`。此时，如果my-volume不存在，那么docker会自动创建my-volume，然后再挂载。
>
> 也可以不指定host上的volume：
>
> ```bash
> docker run -it -v /mydata alpine sh
> ```
>
> 此时docker将自动创建一个匿名的volume，并将其挂载到container中的/mydata目录。匿名volume在host机器上的目录路径类似于：`/var/lib/docker/volumes/300c2264cd0acfe862507eedf156eb61c197720f69e7e9a053c87c2182b2e7d8/_data`。
>
> 除了让docker帮我们自动创建volume，我们也可以自行创建：
>
> ```bash
> docker volume create my-volume-2
> ```
>
> 然后将这个已有的my-volume-2挂载到container中:
>
> ```bash
> docker run -it -v my-volume-2:/mydata alpine sh
> ```
>
> 需要注意的是，与bind mount不同的是，如果volume是空的而container中的目录有内容，那么docker会将container目录中的内容拷贝到volume中，但是如果volume中已经有内容，则会将container中的目录覆盖。请参考[这里](https://medium.com/@yaofei/docker-volume-what-i-learned-27134081d6d9)。
>
> ### Dockerfile中的VOLUME
>
> 在Dockerfile中，我们也可以使用VOLUME指令来申明contaienr中的某个目录需要映射到某个volume：
>
> ```bash
> #Dockerfile
> VOLUME /foo
> ```
>
> 这表示，在docker运行时，docker会创建一个匿名的volume，并将此volume绑定到container的/foo目录中，如果container的/foo目录下已经有内容，则会将内容拷贝的volume中。也即，Dockerfile中的`VOLUME /foo`与`docker run -v /foo alpine`的效果一样。
>
> Dockerfile中的VOLUME使每次运行一个新的container时，都会为其自动创建一个匿名的volume，如果需要在不同container之间共享数据，那么我们依然需要通过`docker run -it -v my-volume:/foo`的方式将/foo中数据存放于指定的my-volume中。
>
> 因此，VOLUME /foo在某些时候会产生歧义，如果不了解的话将导致问题。
>
> 
>
> 作者：无知者云
> 链接：https://www.jianshu.com/p/ef0f24fd0674
> 来源：简书
> 著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

#### B）`Docker Run`

`docker`里的文件夹路径映射到物理机实路径，应该是相当于在`docker`里创建软链接，链接到物理机路径文件夹，文件夹内的数据存储实际上是放在物理机这个路径的。（我的猜测）

`Docker`常见命令：

```bash
# 查看正在运行的容器列表
docker -ps
# or
docker container ls

# 列出所有容器，不论正在运行还是处于停止状态
docker ps -a
# or
docker container ls -a

# 删除容器及容器关联的volumes
docker rm -v containerID(or ContainerName)
# 强制删除正在运行中的容器
docker rm -f containerID(or ContainerName)]

# 查看所有容器卷
docker volume ls
# 查看卷详细信息
docker volume inspect (volumeID or volumeName)
# 单独删除docker volume
docker volume rm volumeID(or volumeName)

# 列出所有映像
docker image ls -a

# 命令行登陆容器内部
docker exec -it containerID(or ContainerName) /bin/bash

# 查看[容器]、[映像]、[网络]等[OPTION]具体信息：
docker inspect [OPTIONS]

# 查看容器内部运行日志
docker logs -f containerID

# 复制宿主机文件到容器内
docker cp [OPTIONS] SRC_PATH containerID:DEST_PATH

# 创建docker网络
docker network create xxx-net
# 举例，创建名叫swag-net的网络，并指定网段（16位掩码）和网关：
sudo docker network create --driver bridge --subnet 172.18.0.0/16 --gateway 172.18.0.1 swag-net
# 只有指定subnet网段（子网掩码），docker容器才可配置成静态IP；

# 查看容器的目录挂载信息
docker inspect -f "{{.Mounts}}" containerID(or ContainerName)

# 升级镜像
docker pull
# 升级具体镜像
docker pull imageID(or Name)
```

#### C）`Docker-Compose`

`Docker-Compose`为一个`project`，里面含有不同的`service`（or 容器），这些`services`组成一个`stack`。

利用`networks`将`services`分配在一个共同的虚拟网桥（网关）中形成局域网，这些容器可以在这个局域网内相互访问。容器内部访问与其同网桥的容器：

```bash
curl http://ServiceName(or ContainerName):port
#或者：
curl http://ContainerIP:port
```

`Docker-compose`命令（均需要在`docker-compose.yml`文件所在路径打开命令行）：

【注意】：`docker-compose`命令是基于`docker-compose.yml`文件执行的，有点类似于配置文件。当对已经安装的`docker-compose app`进行修改或者删除操作时，如果它的`docker-compose.yml`文件在之前已经被修改，那可能会出现操作错误；

```bash
# 停止并删除docker stack（containers）（映射出来的持久化数据不会被删除）；
docker-compose down

# 从映像文件创建新的docker stack（前台创建，安装完退出bash任务会同时STOP这些容器）
# 前台部署完成后，会自动进入docker-compose日志，而且是实时滚动的，方便调试；
docker-compose up
# 从映像文件创建新的docker stack（后台创建）
docker-compose up -d

# 停止该docker stack；
docker-compose stop

# docker-compose日志；
docker-compose logs

# 其他
docker-compose restart
docker-compose start

# 升级所有docker-compose镜像
docker-compose pull
# or 升级具体docker-compose镜像
docker-compose pull stackName
```

【`yml`文件语法】

- 建议用`2.1`版本`yml`语法：`version: '2.1'`；

- 靠空格缩进来区分从属级别；

- 它是`json`文件的超集；

-  `-` 后建议使用双引号 `" "` 将字符串框住，如：`- "swag-net"`、- `"443:443"`等；

- 最后一级且后面不跟参数时可以用 `-` ，也可以用冒号`:` 。但是前面级别要用冒号，如：

  ```yaml
  version: '2.1'
  
  services:
    db:
      imag: mariadb:10.5
      volumes:
        - "/opt/SeafilePro/seafile-mysql/db:/var/lib/mysql"
    
  networks:
    swag-net:
      external: true
  ```

  

- s 

#### D）更换`QTS`系统`docker`的源

**`qpkg`环境**
通常qnap市场中下载的qpkg应用，其环境变量就在自己的包环境中.所以要修改系统中的配置，通常需要修改`qpkg`应用中对应的配置。即`/share/CACHEDEV1_DATA/.qpkg/xxx(app-name)`中的配置

**`container-station` 的`docker`配置**
Ubuntu系统docker的源路径为：`/etc/docker/daemon.json`，与Ubuntu相区别，QTS系统中docker源的位置为`/share/CACHEDEV1_DATA/.qpkg/etc/docker.json`

**国内`Docker`镜像源**
传说中的官方镜像: https://registry.docker-cn.com
网易镜像: http://hub-mirror.c.163.com
阿里云镜像: https://3laho3y3.mirror.aliyuncs.com
`daocloud`镜像: http://f1361db2.m.daocloud.io
腾讯云镜像: https://mirror.ccs.tencentyun.com

这里添加阿里云账户中属于自己的docker镜像加速地址：打开阿里云账户`控制台`，进入`容器镜像服务`，选择`镜像工具`，点开镜像加速器，找到加速器地址并复制

| 我的加速器地址                       |
| ------------------------------------ |
| https://hvzacmbc.mirror.aliyuncs.com |

`root`账户下打开`docker.json`，加入以下代码（如果已有最外层中括号，示例中中括号就不需要添加）：

```json
{
  "registry-mirrors": ["https://hvzacmbc.mirror.aliyuncs.com"]
}
```

**重启container-station生效**

```bash
/etc/init.d/container-station.sh restart
```

#### E）`QTS`系统中的`docker`架构

`docker pull`命令拉取后的`docker`镜像文件默认存放路径：

```bash
/share/CACHEDEV1_DATA/Public/Container/container-station-data/lib/docker/overlay2/
```

*注意：已将`/share/CACHEDEV1_DATA/Public/Container/container-station-data/lib`路径下的`lxd`、`lxc`、`docker`三个文件夹的权限由kingchl仅可执行，变更为kingchl且可读；*

`overlay2`文件夹下的文件夹里的内容，如：

```bash
[/share/CACHEDEV1_DATA/Public/Container/container-station-data/lib/docker/overlay2] # ll
total 44K
drwxr-x--x 11 admin administrators 4.0K 2021-12-19 12:24 ./
drwxr-x--x 14 admin administrators 4.0K 2021-12-14 16:20 ../
drwxr-x--x  3 admin administrators 4.0K 2021-12-19 10:53 12ff21fdc08fd9d78f24091997ba32c1bfe9deb2dfb73a8e8fa4e9d89256c313/
drwxr-x--x  4 admin administrators 4.0K 2021-12-19 10:53 68e9f8c8651dfa24bc0f3723177481dff3dfb090f9ed4f1a561aa07376af9ca3/
drwxr-x--x  5 admin administrators 4.0K 2021-12-19 13:04 e2c0dcfe53553cc8379d7295724c87bfe684bc186fe2d3903c55b9db04f68fb8/
drwxr-x--x  4 admin administrators 4.0K 2021-12-19 12:24 e2c0dcfe53553cc8379d7295724c87bfe684bc186fe2d3903c55b9db04f68fb8-init/
drwxr-x--x  2 admin administrators 4.0K 2021-12-19 12:24 l/

```

进一步查看其中一个，如`e2c0dcfe53553cc8379d7295724c87bfe684bc186fe2d3903c55b9db04f68fb8`

```bash
[/share/CACHEDEV1_DATA/Public/Container/container-station-data/lib/docker/overlay2/e2c0dcfe53553cc8379d7295724c87bfe684bc186fe2d3903c55b9db04f68fb8/diff] # ll
total 88K
drwxr-x--x 11 admin administrators 4.0K 2021-12-19 12:24 ./
drwxr-x--x  5 admin administrators 4.0K 2021-12-19 13:04 ../
drwxr-x--x  2 admin administrators 4.0K 2021-12-19 12:24 data/
drwxr-x--x  4 admin administrators 4.0K 2021-12-19 12:24 dev/
-rwxr-x--x  1 admin administrators  59M 2020-09-17 01:05 docker*
-rwxr-x--x  1 admin administrators  18M 2021-08-18 01:33 docker-compose*
-rwxr-x--x  1 admin administrators    0 2021-12-19 12:24 .dockerenv*
drwxr-x--x  3 admin administrators 4.0K 2021-12-19 12:24 etc/
-rwxr-x--x  1 admin administrators  44M 2021-07-15 02:58 helm*
-rwxr-x--x  1 admin administrators  24M 2020-10-29 02:41 kompose*
-rwxr-x--x  1 admin administrators  42M 2020-03-26 03:20 kubectl*
-rwxr-x--x  1 admin administrators  41M 2021-12-07 13:40 portainer*
drwxr-x--x  2 admin administrators 4.0K 2021-12-19 12:24 proc/
drwxr-x--x  2 admin administrators 4.0K 2021-12-07 13:43 public/
drwxr-x--x  3 admin administrators 4.0K 2021-12-07 13:43 storybook/
drwxr-x--x  2 admin administrators 4.0K 2021-12-19 12:24 sys/
drwxr-x--x  2 admin administrators 4.0K 2021-11-15 04:46 tmp/
drwxr-x--x  3 admin administrators 4.0K 2021-12-19 12:24 var/
```

会发现这个目录结构已经很类似于一个Linux的文件系统根目录了，具体了解：

> [Docker笔记（一）- 镜像与容器，Overlay2](D:\文档中心\【卷·一】理工\【网络与服务器】\集锦\容器技术\Docker笔记（一）- 镜像与容器，Overlay2 - 知乎.mhtml)

#### F）部署前的准备

查看nas的端口占用：

```bash
# 查看系统所有端口占用
netstat -ntlp
# 查看系统所有端口占用并筛出与dockerd字眼相关的
netstat -ntlp | grep dockerd
```

#### G）部署 `portainer-ce`

代码如下（`root`账户登陆`ssh`）：

```bash
#从镜像源拉取portainer-ce镜像（image)
docker pull portainer/portainer-ce
#安装镜像，配置端口、环境变量、映射NAS实体路径，形成容器（container）
docker run -p 9000:9000 -p 8000:8000 --name portainer \
--restart=always \
-v /var/run/docker.sock:/var/run/docker.sock \
-v /share/CACHEDEV1_DATA/Public/Container/Docker/portainer/data:/data \
-d portainer/portainer-ce
```

#### H）部署`Heimdall`

```bash
#从镜像源拉取镜像（image)
docker pull lscr.io/linuxserver/heimdall
#安装Container，注意反斜杠\前面的空格很重要，不然会认为上下行不空格而连写在一起
docker run --name=heimdall -e PUID=1000 -e PGID=1000 -e TZ=Asia/Shanghai \
-p 9002:80 -p 9003:443 \
-v /share/CACHEDEV1_DATA/Public/Container/Docker/Heimdall/config:/config \
--net 'kingchl-net' \
--ip '172.18.0.5' \
--restart always \
-d linuxserver/heimdall:latest
```

`Github`地址：[docker-heimdall](https://github.com/linuxserver/docker-heimdall)

---

最新版测试：

```bash
# 安装Container，注意反斜杠\前面的空格很重要，不然会认为上下行不空格而连写在一起
# 【注意】映射文件夹需要777权限，chmod -R 777 ./Heimdall-ls154，否则无法完成部署；
# 应该不用完全777权限，只要管理员组成员能rwx就行；
docker run --name=Heimdall-ls154 -e PUID=1000 -e PGID=1000 -e TZ=Asia/Shanghai \
-p 9030:80 -p 9031:443 \
-v /share/CACHEDEV1_DATA/Public/Container/Docker/Heimdall-ls154/config:/config \
--net 'kingchl-net' \
--ip '172.18.0.55' \
--restart always \
-d heimdall-2.2.2-ls154:v1.0

# 已安装，但是似乎没必要，回头删除，20220125
```

---

【安装步骤】

1. 根据`docker`自定义网络`kingchl-net`统一部署的静态`IP`地址修改`docker CLI`代码；

2. `docker run`安装部署；

3. 修改`heimdall`的`.env`文件

   ```bash
   # 容器内部路径（注意它是隐藏文件）：
   /config/www/.env
   # or本地持久化路径：
   heimdall/config/www/.env
   
   # 修改URL
   vim .env
   
   APP_URL=http://localhost
   ↓
   APP_URL=https://heimdall.kingchl.cn:9999
   ```

4. 配置`swag`反向代理

   ```nginx
   ## Version 2021/05/18
   # make sure that your dns has a cname set for heimdall
   
   server {
       # 监听swag容器的443端口，此端口映射宿主机（NAS）的8043的端口，并由路由器9999端口转发；
       listen 443 ssl;
       listen [::]:443 ssl;
   
       server_name heimdall.*;
   
       include /config/nginx/ssl.conf;
   
       client_max_body_size 0;
   
       # enable for ldap auth, fill in ldap details in ldap.conf
       #include /config/nginx/ldap.conf;
   
       # enable for Authelia
       #include /config/nginx/authelia-server.conf;
   
       location / {
           # enable the next two lines for http auth
           #auth_basic "Restricted";
           #auth_basic_user_file /config/nginx/.htpasswd;
   
           # enable the next two lines for ldap auth
           #auth_request /auth;
           #error_page 401 =200 /ldaplogin;
   
           # enable for Authelia
           #include /config/nginx/authelia-location.conf;
   
           include /config/nginx/proxy.conf;
           include /config/nginx/resolver.conf;
           set $upstream_app heimdall;
           set $upstream_port 443;
           set $upstream_proto https;
           proxy_pass $upstream_proto://$upstream_app:$upstream_port;
   
       }
   }
   ```

【注意事项】

- 目前有个`bug`：登陆页面正确输入账号密码登陆后`heimdall`重定向出现丢失端口`9999`的问题，此时刷新下`URL`地址即可；

- 该容器内部关键路径：

  ```bash
  # 配置文件路径
  /config
  # heimdall站点文件路径
  /var/www/localhost/heimdall
  ```

- 【定期清理】另外，此`config/log/heimdall/laravel.log`日志文件有点大。。2个G了；

- > ## New background image not being set
  >
  > If you are using the docker image（镜像） or a default php install you may find images（背景图片） over 2MB wont get set as the background image, you just need to change the `upload_max_filesize` in the php.ini.
  >
  > If you are using the linuxserver.io docker image simply edit `/path/to/config/php/php-local.ini` and add `upload_max_filesize = 30M` to the end.

#### I）部署`Calibre-Web`

利用`docker-compose`进行拉取和安装：

```yml
version: "2.1"
services:
  calibre-web:
    image: lscr.io/linuxserver/calibre-web
    container_name: calibre-web
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Shanghai
      - DOCKER_MODS=linuxserver/calibre-web:calibre #optional
      - OAUTHLIB_RELAX_TOKEN_SCOPE=1 #optional
    volumes:
      - /share/CACHEDEV1_DATA/Public/Container/Docker/calibre-web/data:/config
      - /share/CACHEDEV3_DATA/动态数据存储/归档中心/文档中心/【卷·零】图书馆/Calibre书库:/books
    ports:
      - 9004:8083
    restart: unless-stopped
```

终端的路径指向`docker-compose.yml`所在文件夹，非root账户状态下运行：

```bash
sudo docker-compose up
```

注意创建的`Docker/calibre-web/data`文件夹权限，要设置成执行该安装的用户可写、可执行；

安装时间比较久，下载好后执行安装估计要半小时。

#### J）部署WizNote

`Docker-Hub`：[wiznote/wizserver](https://registry.hub.docker.com/r/wiznote/wizserver)，`GitHub`：[WizTeam](https://github.com/WizTeam)（服务端非开源）

```bash
docker run --name wiz -it -d \
-v /share/CACHEDEV3_DATA/DYNAMIC/ARCHIVE/Docker/wiznote/wizdata:/wiz/storage \
-v /etc/localtime:/etc/localtime \
-p 9006:80 -p 9007:9269/udp \
--restart=always \
--net 'kingchl-net' \
--ip '172.18.0.40' \
wiznote/wizserver
```

~~初始管理员账号：admin@wiz.cn，密码：123456~~。已修改为：管理员账号：604378841@qq.com，密码：;

免费版限5用户；

`swag`反向代理配置文件：

```nginx
## Version 2021/05/18
# make sure that your dns has a cname set for wiznote and that your wiznote container is named wiznote

server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name wiznote.*;

    include /config/nginx/ssl.conf;

    client_max_body_size 0;

    # enable for ldap auth, fill in ldap details in ldap.conf
    #include /config/nginx/ldap.conf;

    # enable for Authelia
    #include /config/nginx/authelia-server.conf;
	
	proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header x-wiz-real-ip $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_set_header X-Forwarded-Proto $scheme;

    location / {
        #enable the next two lines for http auth
        #auth_basic "Restricted";
        #auth_basic_user_file /config/nginx/.htpasswd;

        # enable the next two lines for ldap auth
        #auth_request /auth;
        #error_page 401 =200 /ldaplogin;

        # enable for Authelia
        #include /config/nginx/authelia-location.conf;

        #include /config/nginx/proxy.conf;
        include /config/nginx/resolver.conf;
        set $upstream_app wiz;
        set $upstream_port 80;
        set $upstream_proto http;
        proxy_pass $upstream_proto://$upstream_app:$upstream_port;

    }
}
```

【注意事项】

1. 配置反向代理：

   > [为知笔记 | 为知笔记私有部署配置nginx反向代理和https的方法 (wiz.cn)](https://www.wiz.cn/zh-cn/docker-https)
   >
   > ## 使用nginx反向代理需要配置proxy_set_header
   >
   > ```nginx
   >    server {
   >         proxy_http_version 1.1;
   >         proxy_set_header Upgrade $http_upgrade;
   >         proxy_set_header Connection "upgrade";
   >         proxy_set_header X-Real-IP $remote_addr;
   >         proxy_set_header x-wiz-real-ip $remote_addr;
   >         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
   >         proxy_set_header Host $http_host;
   >         proxy_set_header X-Forwarded-Proto $scheme;
   >         ...
   > ```

2. 添加SMTP邮件功能，可使用邮箱找回密码：

   ```bash
   # SMTP服务器：
   smtp.qq.com
   # 启用SSL连接，端口：
   465
   # 用户名：
   345290729@qq.ccom
   # token（第三方登录授权码）
   aymaufqctrnqbhed
   # 默认发件人邮箱
   avrsilmx@qq.com (or 345290729@qq.ccom)
   ```

2. 

#### K）部署`BookStack`

官网地址：[bookstack](https://www.bookstackapp.com/)，安装地址：[linuxserver](https://github.com/linuxserver)/**[docker-bookstack](https://github.com/linuxserver/docker-bookstack)**；

另一个MDwiki做备选：[Dynalon](https://github.com/Dynalon)/**[mdwiki](https://github.com/Dynalon/mdwiki)**

##### a）利用`docker-compose`部署

```yml
---
version: "2"
services:
  bookstack:
    image: lscr.io/linuxserver/bookstack:latest
    container_name: bookstack
    environment:
      - PUID=1000
      - PGID=1000
      # 必须此处指定APP_URL，若后续在.env中指定，则无法生效。另外刚安装完可能是因为还在后台部署安装，又或者是因为浏览器缓存没及时更新。短时间内访问其地址时，在自动重定向到的/login页面中提示重定向次数过多，等待几分钟即可；
      # 如果域名发生变更，建议备份好数据文件后，删除干净此容器及映射文件夹，配置好新的APP_URL然后重新安装。
      - APP_URL=https://bookstack.kingchl.cn:9999
      - DB_HOST=bookstack_db
      - DB_USER=bookstack
      - DB_PASS=cheng5969
      - DB_DATABASE=bookstackapp
    volumes:
      - /share/CACHEDEV3_DATA/DYNAMIC/ARCHIVE/Docker/bookstack/bookstack-data:/config
    ports:
      - 9008:80
    restart: unless-stopped
    networks:
      kingchl-net:
        ipv4_address: 172.18.0.30
    depends_on:
      - bookstack_db
      
  bookstack_db:
    image: lscr.io/linuxserver/mariadb
    container_name: bookstack_db
    environment:
      - PUID=1000
      - PGID=1000
      - MYSQL_ROOT_PASSWORD=cheng5969
      - TZ=Asia/Shanghai
      - MYSQL_DATABASE=bookstackapp
      - MYSQL_USER=bookstack
      - MYSQL_PASSWORD=cheng5969
    volumes:
      - /share/CACHEDEV3_DATA/DYNAMIC/ARCHIVE/Docker/bookstack/bookstack_db-data:/config
    restart: unless-stopped
    networks:
      kingchl-net:
        ipv4_address: 172.18.0.31

networks:
  kingchl-net:
    external: true
```

终端的路径指向`docker-compose.yml`所在文件夹，root账户下运行：

```bash
docker-compose up
```

注意创建的相关文件夹权限，要设置成所需登陆的用户可写、可执行；

初始账户密码：ID : admin@admin.com  pswd: password ;

##### b）利用宝塔web服务器安装

`Github`地址：[BookStackApp](https://github.com/BookStackApp)/**[BookStack](https://github.com/BookStackApp/BookStack)**

【安装步骤】【本`NAS`最终采用`docker`安装】

**Requirements:**

BookStack has the following requirements:

- PHP >= 7.3
  - For installation and maintenance, you’ll need to be able to run `php` from the command line.
  - Required Extensions: *OpenSSL, PDO, MBstring, Tokenizer, GD, MySQL, SimpleXML & DOM*
- MySQL >=  5.6 or MariaDB >= 10.0
  - Single Database *(All permissions advised since application manages schema)*
- Git Version Control
  - (Not strictly required but helps manage updates)
- **[Composer](https://getcomposer.org/)** >= v2.0

**Manual Installation:**

Ensure the above requirements are met before installing.

This project currently uses the `release` branch of the BookStack GitHub repository as a stable channel for providing updates. The installation is currently somewhat complicated and will be made simpler in future releases. Some PHP or Laravel experience will make this easier.

1. Clone the release branch of the BookStack GitHub repository into a folder.

   ```bash
   git clone https://github.com/BookStackApp/BookStack.git --branch release --single-branch
   ```

2. 更新composer版本到最新：

   ```bash
   # 查看composer版本
   composer --version
   # 执行更新
   /usr/bin/composer self-update
   ```

3. `cd` into the application folder and run `composer install --no-dev`.

4. Copy the `.env.example` file to `.env` and fill with your own database and mail details.

5. Ensure the `storage`, `bootstrap/cache` & `public/uploads` folders are writable by the web server.

6. In the application root, Run `php artisan key:generate` to generate a unique application key.

7. If not using Apache or if `.htaccess` files are disabled you will have to create some URL rewrite rules as shown below.

8. Set the web root on your server to point to the BookStack `public` folder. This is done with the `root` setting on Nginx or the `DocumentRoot` setting on Apache.

9. Run `php artisan migrate` to update the database.

10. Done! You can now login using the default admin details `admin@admin.com` with a password of `password`. You should change these details immediately after logging in for the first time.

11. Webserver Configuration

    - [Example Apache VirtualHost configuration](https://github.com/BookStackApp/devops/blob/main/config/apache/bookstack.conf)
    - [Example Nginx Server block](https://github.com/BookStackApp/devops/blob/main/config/nginx)

12. ```nginx
    # 源自网络
    
    server {
      listen 80;
      server_name 192.168.50.61;
      root /www/wwwroot/bookstack.kingchl.cn/public;
    
      access_log  /www/wwwlogs/bookstack.kingchl.cn.log;
      error_log  /www/wwwlogs/bookstack.kingchl.cn.error.log;
    
      client_max_body_size 1G;
      fastcgi_buffers 64 4K;
    
      index  index.php;
    
      location / {
        try_files $uri $uri/ /index.php?$query_string;
      }
    
      location ~ ^/(?:\.htaccess|data|config|db_structure\.xml|README) {
        deny all;
      }
    
      location ~ \.php(?:$|/) {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
        fastcgi_pass 127.0.0.1:9000;
      }
    
      location ~* \.(?:jpg|jpeg|gif|bmp|ico|png|css|js|swf)$ {
        expires 30d;
        access_log off;
      }
    }
    
    ```

    

【注意事项】

- 关于`URL`：

  ```bash
  #Application URL
  #This must be the root URL that you want to host BookStack on.
  #All URLs in BookStack will be generated using this value
  #to ensure URLs generated are consistent and secure.
  #If you change this in the future you may need to run a command
  #to update stored URLs in the database. Command example:
  #`php artisan bookstack:update-url https://old.example.com https://new.example.com`
  APP_URL=https://example.com
  ```

- 数据库账号资料：

  数据库名：**bookstack_db**，用户：**bookstack_db**，密码：**cheng5969**；

- 目前，`bookstack`全文检索需要将关键字用英文`""`括起来；

- > 实现全文检索。因版本过早，仅供参考；
  >
  > Hi,all the guys,I fixed this problem in v0.28.3.Just add a '%' in SearchService.php.
  > In detail.
  > in \app\Entities\SearchService.php,about line 196.
  > modify
  > $query->orWhere('term', 'like', $inputTerm . '%');
  > to
  > $query->orWhere('term', 'like', '%'.$inputTerm . '%');
  > Just try.

- 



#### L）部署`SeafilePro`

更新日志及`seafile`文档：[Seafile Admin Manual](https://manual.seafile.com/changelog/changelog-for-seafile-professional-server/)；`GitHub`：[haiwen](https://github.com/haiwen)/**[seafile-admin-docs](https://github.com/haiwen/seafile-admin-docs)**；

`SeafilePro`的镜像文件采用`Ubuntu`最小系统制作，支持`apt`，能够安装`tree`、`ranger`等软件；

采用`docker-compose`方式部署（本次部署使用`root`账户），版本`8.0.14`，但是还是按照`seafile9.0`的安装要求，<u>手动建立elasticsearch路径并赋予777权限</u>；

> ### 下载并修改 docker-compose.yml
>
> Seafile 7.1 到 8.0 版本下载 URL： ﻿[docker-compose.yml](https://docs.seafile.com/d/cb1d3f97106847abbf31/files/?p=/docker/pro-edition/docker-compose.yml)﻿
>
> Seafile 9.0 及以后版本下载URL：  ﻿[docker-compose.yml ](https://cloud.seafile.com/f/59462f270674467bb2eb/)	<!--官网下载的这个yml版本中，seafile版本并不是9.0以后版本，目前是8.0.14-->
>
> *9.0 版本升级参考：[Upgrade notes for 9.0](https://manual.seafile.com/upgrade/upgrade_notes_for_9.0.x/)*；
>
> 示例文件到您的服务器上，然后根据您的实际环境修改该文件。尤其是以下几项配置：
>
> - MySQL root 用户的密码 (MYSQL_ROOT_PASSWORD and DB_ROOT_PASSWD)
> - 持久化存储 MySQL 数据的 volumes 目录 (volumes)
> - 持久化存储 Seafile 数据的 volumes 目录 (volumes)
> - 持久化存储 Elasticsearch 索引数据的 volumes 目录 (volumes)
>
> **注意：seafile 9.0 版本，需要手动在宿主机上创建 elasticsearch 的映射路径，并且给 777 权限，否则 elasticsearch 启动会报路径权限问题，命令如下**
>
> ```bash
> mkdir -p /opt/seafile-elasticsearch/data  && chmod 777 -R /opt/seafile-elasticsearch/data
> ```
>
> 具体在我的威联通TS-264C里就是：
>
> ```bash
>  mkdir -p /share/CACHEDEV1_DATA/Public/Container/Docker/SeafilePro/seafile-elasticsearch/data  && chmod 777 -R /share/CACHEDEV1_DATA/Public/Container/Docker/SeafilePro/seafile-elasticsearch/data
> ```
>
> ### 启动 Seafile 服务
>
> 执行以下命令启动 Seafile 服务
>
> ```bash
> docker-compose up
> ```
>
> 如果需要重启这个容器：
>
> ```bash
> docker-compose restart
> ```
>
> **注意：您应该在** `docker-compose.yml`**文件所在的目下执行以上命令。**
>
> ### 查找日志
>
> Seafile 容器中 Seafile 服务本身的日志文件存放在 `/shared/logs/seafile` 目录下，或者您可以在宿主机上 Seafile 容器的卷目录中找到这些日志，例如：`/opt/seafile-data/logs/seafile`
>
> 同样 Seafile 容器的系统日志存放在 `/shared/logs/var-log` 目录下，或者宿主机目录 `/opt/seafile-data/logs/var-log`。
>
> ## Seafile 目录结构
>
> ### `/shared`
>
> 共享卷的挂载点,您可以选择在容器外部存储某些持久性信息.在这个项目中，我们会在外部保存各种日志文件和上传数据。 这使您可以轻松重建容器而不会丢失重要信息。
>
> - /shared/seafile: Seafile 服务的配置文件，日志文件以及数据文件
>   - /shared/seafile/logs: Seafile 服务运行产生的日志文件目录。比如您可以在 `/shared/seafile/logs/seafile.log` 文件中看到 seaf-server 的日志
>   - /shared/seafile/seafile-data: 如果您没有配置S3或者OSS等对象存储，那么用户上传的数据将会存放到该目录下。
> - /shared/logs: 日志目录
>   - /shared/logs/var-log: 我们将容器内的`/var/log`链接到本目录。您可以在`/shared/logs/var-log/nginx/`中找到 nginx 的日志文件
> - /shared/ssl: 存放证书的目录，默认不存在
>
> ### 升级 Seafile 服务
>
> 如果要升级 Seafile 服务到最新版本：
>
> ```bash
> docker pull docker.seafile.top/seafileltd/seafile-pro-mc:latest
> docker-compose down
> docker-compose up -d
> ```
>
> ## 备份和恢复
>
> ### 目录结构
>
> 我们假设您的 seafile 数据卷路径是 `/opt/seafile-data`，并且您想将备份数据存放到 `/opt/seafile-backup` 目录下。
>
> 您可以创建一个类似以下 `/opt/seafile-backup` 的目录结构：
>
> ```bash
> /opt/seafile-backup---- databases/  用来存放 MySQL 容器的备份数据---- data/  用来存放 Seafile 容器的备份数据
> ```
>
> 要备份的数据文件：
>
> ```bash
> /opt/seafile-data/seafile/conf  # configuration files/opt/seafile-data/seafile/seafile-data # data of seafile/opt/seafile-data/seafile/seahub-data # data of seahub
> ```
>
> ### 备份数据
>
>  步骤：
>
> 1. 备份 MySQL 数据库数据；
> 2. 备份 Seafile 数据目录；
>
> - 备份数据库：
>
>   ```bash
>   # 建议每次将数据库备份到一个单独的文件中。至少在一周内不要覆盖旧的数据库备份。cd /opt/seafile-backup/databasesdocker exec -it seafile-mysql mysqldump  -uroot --opt ccnet_db > ccnet_db.sqldocker exec -it seafile-mysql mysqldump  -uroot --opt seafile_db > seafile_db.sqldocker exec -it seafile-mysql mysqldump  -uroot --opt seahub_db > seahub_db.sql
>   ```
>
> - 备份 Seafile 资料库数据：
>
>   - 直接复制整个数据目录
>
>     ```bash
>     cp -R /opt/seafile-data/seafile /opt/seafile-backup/data/cd /opt/seafile-backup/data && rm -rf ccnet
>     ```
>
>   - 使用 rsync 执行增量备份
>
>     ```bash
>     rsync -az /opt/seafile-data/seafile /opt/seafile-backup/data/cd /opt/seafile-backup/data && rm -rf ccnet
>     ```
>
> ### 恢复数据
>
> - 恢复数据库：
>
>   ```bash
>   docker cp /opt/seafile-backup/databases/ccnet_db.sql seafile-mysql:/tmp/ccnet_db.sqldocker cp /opt/seafile-backup/databases/seafile_db.sql seafile-mysql:/tmp/seafile_db.sqldocker cp /opt/seafile-backup/databases/seahub_db.sql seafile-mysql:/tmp/seahub_db.sql﻿
>   docker exec -it seafile-mysql /bin/sh -c "mysql -uroot ccnet_db < /tmp/ccnet_db.sql"docker exec -it seafile-mysql /bin/sh -c "mysql -uroot seafile_db < /tmp/seafile_db.sql"docker exec -it seafile-mysql /bin/sh -c "mysql -uroot seahub_db < /tmp/seahub_db.sql"
>   ```
>
> - 恢复 seafile 数据：
>
>   ```bash
>   cp -R /opt/seafile-backup/data/* /opt/seafile-data/seafile/
>   ```
>
> ## 垃圾回收
>
> 在 seafile 中，当文件被删除时，组成这些文件的块数据不会立即删除，因为可能有其他文件也会引用这些块数据(用于去重功能的实现)。为了真正删除无用的块数据，还需要额外运行"﻿[GC](https://manual-cn.seafile.com/maintain/seafile_gc.html)﻿"程序。GC 会自动检测到哪些数据块不再被任何文件所引用，并清除它们。
>
> GC 脚本被放在docker容器的 `/scripts` 目录下。执行 GC 的方法很简单：`docker exec seafile /scripts/gc.sh`。对于社区版来说，该程序会暂停 Seafile 服务，但这是一个相对较快的程序，一旦程序运行完成，Seafile 服务也会自动重新启动。而专业版提供了在线运行 GC 的功能，不会暂停 Seafile 服务。
>
> ## 问题排查
>
> 您可以运行 `docker exec` 之类的docker命令来查找错误。
>
> ```bash
> docker exec -it seafile /bin/bash
> ```
>
> **其他命令摘录：**
>
> 重启 Seafile 和 Seahub ：
>
> ```bash
> ./seahub.sh restart
> ./seafile.sh restart
> ```
>
> 源自：[用 Docker 部署 Seafile 专业版]([用Docker部署Seafile - Seafile Cloud](https://cloud.seafile.com/published/seafile-manual-cn/docker/pro-edition/用Docker部署Seafile.md))

针对我的威联通`TS-264C`，`yml`配置文件修改如下：

> -------此处`yml`文件仅供存档。为配合反向代理软件，正式在`TS-264C`上部署的`yml`文件在下文列出；------
>
> ```yaml
> version: '2.0'
> services:
>   db:
>     image: mariadb:10.5
>     restart: always
>     container_name: seafile-mysql
>     environment:
>       - MYSQL_ROOT_PASSWORD=cheng5969  # Requested, set the root's password of MySQL service.
>       - MYSQL_LOG_CONSOLE=true
>     volumes:
>       - /share/CACHEDEV1_DATA/Public/Container/Docker/SeafilePro/seafile-mysql/db:/var/lib/mysql  # Requested, specifies the path to MySQL data persistent store.
>     networks:
>       - seafile-net
> 
>   memcached:
>     image: memcached:1.5.6
>     restart: always
>     container_name: seafile-memcached
>     entrypoint: memcached -m 256
>     networks:
>       - seafile-net
> 
>   elasticsearch:
>     image: elasticsearch:6.8.20
>     restart: always
>     container_name: seafile-elasticsearch
>     environment:
>       - discovery.type=single-node
>       - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
>     volumes:
>       - /share/CACHEDEV1_DATA/Public/Container/Docker/SeafilePro/seafile-elasticsearch/data:/usr/share/elasticsearch/data  # Requested, specifies the path to Elasticsearch data persistent store.
>     networks:
>       - seafile-net
>          
>   seafile:
>     # image: docker.seafile.top/seafileltd/seafile-pro-mc:latest
>     image: docker.seafile.top/seafileltd/seafile-pro-mc:8.0.14
>     restart: always
>     container_name: seafile
>     ports:
>       - "9010:80"
> #     - "443:443"  # If https is enabled, cancel the comment.
>     volumes:
>       - /share/CACHEDEV1_DATA/Public/Container/Docker/SeafilePro/seafile-data:/shared   # Requested, specifies the path to Seafile data persistent store.
>     environment:
>       - DB_HOST=db
>       - DB_ROOT_PASSWD=cheng5969  # Requested, the value shuold be root's password of MySQL service.
>       - TIME_ZONE=Asia/Shanghai # Optional, default is UTC. Should be uncomment and set to your local time zone.
>       - SEAFILE_ADMIN_EMAIL=345290729@qq.com # Specifies Seafile admin user, default is 'me@example.com'
>       - SEAFILE_ADMIN_PASSWORD=cheng5969    # Specifies Seafile admin password, default is 'asecret'
>       - SEAFILE_SERVER_LETSENCRYPT=false   # Whether to use https or not
>       - SEAFILE_SERVER_HOSTNAME=seafile.example.com # Specifies your host name if https is enabled
>     depends_on:
>       - db
>       - memcached
>       - elasticsearch
>     networks:
>       - seafile-net
> 
> networks:
>   seafile-net:
> ```
>
> -------此处`yml`文件仅供存档。为配合反向代理软件，正式在`TS-264C`上部署的`yml`文件在下文列出；------

【安装步骤】

1. 统一`docker networks`为：`kingchl-net`，

   使用 `docker-compose`或者 `docker run`命令创建容器时，如果需要指定容器的静态IP地址，则事先需要创建自定义网桥，使用`--subnet`命令来指定网段和子网掩码，`--gateway`来指定网关；

   ```bash
   docker network create --driver bridge --subnet 172.18.0.0/16 --gateway 172.18.0.1 kingchl-net
   ```

2. 在安装`seafile pro`之前，一定要做两件事，否则`elasticsearch`无法搜索：

   - 手动建立文件夹`seafile-elasticsearch/data`并赋予`777`权限，否则在部署`seafile pro`时，`elasticsearch`在`data`文件夹里无法创建任何文件，导致`seafile pro`搜索功能失效；

     ```bash
     mkdir -p /share/CACHEDEV3_DATA/DYNAMIC/ARCHIVE/Docker/SeafilePro/seafile-elasticsearch/data  && chmod 777 -R /share/CACHEDEV3_DATA/DYNAMIC/ARCHIVE/Docker/SeafilePro/seafile-elasticsearch/data
     ```

   - 【2022/01/22补充】这里建议手动创建`SeafilePro`文件夹，并将其整个文件夹赋予777权限，然后再执行`docker-compose up`，并在部署成功后，再执行一便赋予`777`权限（包含子文件夹）；

   - `seafile`版本：`8.0.14`，`elasticsearch`版本：`6.8.20`，跟这个可能也有关系；

   - 经实践测试，搜索功能正常，历史版本功能也正常；

3. 修改官方`seafile pro`的`yml`文件：

   - 修改`docker-compose.yml`文件版本为`2.1`
   - 配置挂载目录，适应威联通`TS-264C`；
   - 配置数据库密码；
   - 配合`swag`（`Nginx`反向代理）设置容器静态IP；
   - 静态路由表参照：`威联通NAS笔记/反向代理与域名/实现方案/实现反向代理及SSL证书`；
   - `yml`如下（正式部署）：

   ```yaml
   version: '2.1'
   services:
     db:
       image: mariadb:10.5
       restart: always
       container_name: seafile-mysql
       environment:
         - MYSQL_ROOT_PASSWORD=cheng5969  # Requested, set the root's password of MySQL service.
         - MYSQL_LOG_CONSOLE=true
       volumes:
         - /share/CACHEDEV3_DATA/DYNAMIC/ARCHIVE/Docker/SeafilePro/seafile-mysql/db:/var/lib/mysql  # Requested, specifies the path to MySQL data persistent store.
       networks:
         kingchl-net:
           ipv4_address: 172.18.0.11
   
     memcached:
       image: memcached:1.5.6
       restart: always
       container_name: seafile-memcached
       entrypoint: memcached -m 256
       networks:
         kingchl-net:
           ipv4_address: 172.18.0.12
   
     elasticsearch:
       image: elasticsearch:6.8.20
       restart: always
       container_name: seafile-elasticsearch
       environment:
         - discovery.type=single-node
         - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
       volumes:
         - /share/CACHEDEV3_DATA/DYNAMIC/ARCHIVE/Docker/SeafilePro/seafile-elasticsearch/data:/usr/share/elasticsearch/data  # Requested, specifies the path to Elasticsearch data persistent store.
       networks:
         kingchl-net:
           ipv4_address: 172.18.0.13
            
     seafile:
       # image: docker.seafile.top/seafileltd/seafile-pro-mc:latest
       image: docker.seafile.top/seafileltd/seafile-pro-mc:8.0.14
       restart: always
       container_name: seafile
       ports:
         - "9010:80"
         - "443:443"  # If https is enabled, cancel the comment.
       volumes:
         - /share/CACHEDEV3_DATA/DYNAMIC/ARCHIVE/Docker/SeafilePro/seafile-data:/shared   # Requested, specifies the path to Seafile data persistent store.
       environment:
         - DB_HOST=db
         - DB_ROOT_PASSWD=cheng5969  # Requested, the value shuold be root's password of MySQL service.
         - TIME_ZONE=Asia/Shanghai # Optional, default is UTC. Should be uncomment and set to your local time zone.
         - SEAFILE_ADMIN_EMAIL=345290729@qq.com # Specifies Seafile admin user, default is 'me@example.com'
         - SEAFILE_ADMIN_PASSWORD=cheng5969    # Specifies Seafile admin password, default is 'asecret'
         - SEAFILE_SERVER_LETSENCRYPT=false   # Whether to use https or not
         - SEAFILE_SERVER_HOSTNAME=seafile.example.com # Specifies your host name if https is enabled
       depends_on:
         - db
         - memcached
         - elasticsearch
       networks:
         kingchl-net:
           ipv4_address: 172.18.0.10
   
   networks:
     kingchl-net:
       external: true
   ```

4. 反向代理`seafile pro`的`swag`配置文件：

   ```nginx
   ## Version 2021/05/18
   # For use with the official Seafile Docker image (https://download.seafile.com/published/seafile-manual/docker/deploy%20seafile%20with%20docker.md)
   # Requires that the seafile container uses the following env variables:
   #     SEAFILE_SERVER_LETSENCRYPT=true
   #     SEAFILE_SERVER_HOSTNAME=seafile.yourdomain.com
   # Restart or create the seafile container after enabling this subdomain and restarting the letsencrypt contianer
   
   server {
       # 监听swag容器的443端口，此端口映射宿主机（NAS）的8043的端口，并由路由器9999端口转发；
       listen 443 ssl;
       listen [::]:443 ssl;
   
       server_name seafile.kingchl.cn;
   
       include /config/nginx/ssl.conf;
   
       client_max_body_size 0;
   
       location / {
           #include /config/nginx/proxy.conf;
           #include /config/nginx/resolver.conf;
           set $upstream_app 172.18.0.10;
           set $upstream_port 80;
           set $upstream_proto http;
           proxy_pass $upstream_proto://$upstream_app:$upstream_port;
   
       }
   }
   ```

   配置文件路径：`/config/nginx/proxy-confs/seafile.subdomain.conf`（`swag`容器内部）；

   【注意】：`seafile pro`有内建的`nginx`反向代理（容器内配置文件路径：`/shared/nginx/conf/seafile.nginx.conf`），已经配置好各个`location`（l`ocation / {}`，`location /seafhttp {}`等），因此在`swag`中的反向代理配置文件就相对简单，直接转发到`80`端口即可；

5. 【待解决】：`http`强制转到`https`；

   【插曲】：在倒腾强制转`https`失败，恢复以前的单`https`配置文件并重启`swag`后，出现当前浏览器打不开`seafile`页面的情况，原因在于浏览器缓存了之前各种不正确重定向的缓存（`Cookie`？），重启当前浏览器或换个浏览器登陆`seafile`能够正常访问；

   ```http
   # Chrome会重定向至下面链接，然后提示“页面不存在”
   https://seafile.kingchl.cn:9999/redirect.html?count=0.33832682748385
   ```

6. 添加`SMTP`邮件服务：

   ```bash
   # 首先载qq邮箱中开启smtp服务，并生成第三方登录授权码：
   aymaufqctrnqbhed
   # seafile-data/seafile/conf/seahub_settings.py中加入如下代码
   EMAIL_USE_TLS = False
   EMAIL_HOST = 'smtp.qq.com'
   EMAIL_HOST_USER = 'avrsilmx@qq.com'
   EMAIL_HOST_PASSWORD = 'aymaufqctrnqbhed'
   EMAIL_PORT = '25'
   DEFAULT_FROM_EMAIL = EMAIL_HOST_USER
   SERVER_EMAIL = EMAIL_HOST_USER
   # 重启docker-compose，如果更改没有生效，请删除seahub_setting.pyc缓存文件.
   docker-compose retart
   # 至此，可以使用邮箱重置用户密码
   ```

7. 嵌入`onlyoffice`，需注意，在`seafile`是`https`访问的前提下，当调用`http`协议的`onlyoffice`时，调用页面会显示空白。所以协议要相一致。调试时可以使用`http`协议登陆`seafile`；

   参考手册：[OnlyOffice](https://manual.seafile.com/deploy/only_office/)、[OnlyOffice 集成](https://www.bookstack.cn/read/Seafile-manual/44.md#%E9%80%9A%E8%BF%87%E5%AD%90%E5%9F%9F%E9%83%A8%E7%BD%B2DocumentServer)；

   ```bash
   # seahub_settings.py中添加如下信息
   
   # Enable OnlyOffice
   ENABLE_ONLYOFFICE = True
   VERIFY_ONLYOFFICE_CERTIFICATE = True
   ONLYOFFICE_APIJS_URL = 'https://oods.kingchl.cn:9999/web-apps/apps/api/documents/api.js'
   ONLYOFFICE_FILE_EXTENSION = ('doc', 'docx', 'ppt', 'pptx', 'xls', 'xlsx', 'odt', 'fodt', 'odp', 'fodp', 'ods', 'fods')
   ONLYOFFICE_EDIT_FILE_EXTENSION = ('docx', 'pptx', 'xlsx')
   
   # Enable force save to let user can save file when he/she press the save button on OnlyOffice file edit page.
   ONLYOFFICE_FORCE_SAVE = True
   ```

8. 开启`wiki`

   ```python
   # 路径：seafile-data/seafile/conf/seahub_settings.py
   # Enable the wiki
   ENABLE_WIKI = True
   ```

6. 

【注意事项】

1. 删除索引并重建：

   命令行进入`seafile`容器后，

   ```bash
   # 进入pro.py所在目录
   cd /opt/seafile/seafile-pro-server-8.0.14/pro
   # 删除索引（可能要求root权限）
   ./pro.py search --clear
   # 重建（更新）索引
   ./pro.py search --update
   ```

   更新索引后，如有新文件被检索到，可以在命令行看到如下类似信息：

   ```bash
   ...
   02/17/2022 23:58:02 [INFO] seafes:181 extract: extracting f39a36d0-e7a4-4c42-b772-367547409634 /家庭服务器技术/Linux系统笔记.md...
   02/17/2022 23:58:02 [INFO] seafes:183 extract: successfully extracted /家庭服务器技术/Linux系统笔记.md
   ...
   Index updated, statistic report:
   
   02/17/2022 23:58:02 [INFO] seafes:165 start_index_local: [commit read] 2
   02/17/2022 23:58:02 [INFO] seafes:166 start_index_local: [dir read]    4
   02/17/2022 23:58:02 [INFO] seafes:167 start_index_local: [file read]   1
   02/17/2022 23:58:02 [INFO] seafes:168 start_index_local: [block read]  1
   ```

   【注意】：文本文件的大小达到100k以上时，`seafile`在建立全文检索的索引时会出错，提示超出大小限制，全文内容无法被检索到，只能检索到该文件的标题。解决办法未知；

   非`docker`版索引文件所在路径为`pro-data/search`，`docker`版的路径为（猜测）：

   ```bash
   /opt/seafile/seafile-pro-server-8.0.14/pro/elasticsearch
   ```

2. 很多`*.sh`文件都在以下目录，包括常用的`seafile.sh`、`seahub.sh`

   ```bash
   /opt/seafile/seafile-pro-server-8.0.14/
   ```

3. 容器关键路径

   ```bash
   # 配置文件位置
   /opt/seafile/conf
   # 日志位置
   /opt/seafile/logs
   # web服务器文件位置，各类*sh执行脚本
   /opt/seafile/seafile-pro-server-8.0.14
   # pro版本的elasticsearch及索引操作脚本pro.py位置
   /opt/seafile/seafile-pro-server-8.0.14/pro
   # 用户数据位置
   /opt/seafile/seafile-data
   # 进程pid
   /opt/seafile/pids
   ```

4. 服务器及客户端更新参考：[Seafile Admin Manual](https://manual.seafile.com/changelog/changelog-for-seafile-professional-server/)、中文版：[`seafile`手册](https://cloud.seafile.com/published/seafile-manual-cn/maintain/seafile_fsck.md)

5. `5.0` 版开始，建议通过 `Web` 界面来修改，不要直接修改 `ccnet.conf` 中的值。`9.0` 版开始 `SERVICE_URL` 需要写到 `seahub_settings.py` 中`SERVICE_URL`=http://www.example.com:8000

6. 


#### M）部署可道云文件管理器

【注意】：使用`kodexplorer`来接管`NAS`中的文件夹数据，会污染`NAS`文件夹以及数据的所有者（所有者会变成一个叫`82` 的用户）及权限；

`kod`官方现在推行`kodbox`，而`kodexplorer`则部署在私人服务器上作服务器文件管理用。这里用的免费版，能增加用户，但不能限制用户空间大小。

由于官方没有`docker`镜像，在`docker-hub`里选择第三方制作的镜像，这里选择`baiyuetribe/kodexplorer`。

安装及路径映射如下：

> （此处仅作记录）
>
> 废旧第一版：
>
> ```bash
> docker run -d -p 9012:80 --name kodexplorer \
> -v /share/CACHEDEV1_DATA:/var/www/html/data/User/admin/TS-264C/CACHEDEV1_DATA \
> -v /share/CACHEDEV3_DATA:/var/www/html/data/User/admin/TS-264C/CACHEDEV3_DATA \
> -v /share/CACHEDEV3_DATA/动态数据存储/开发中心/KodExplorer/admin:/var/www/html/data/User/admin/home \
> -v /share/CACHEDEV3_DATA/动态数据存储/开发中心/KodExplorer/fiona:/var/www/html/data/User/fiona/home \
> -v /share/CACHEDEV3_DATA/动态数据存储/开发中心/KodExplorer/default:/var/www/html/data/User/default/home \
> --restart always \
> baiyuetribe/kodexplorer
> ```
>
> --------------------------------
>
> 废旧第二版
>
> ```bash
> docker run -d -p 9012:80 --name kodexplorer \
> -v /share/CACHEDEV3_DATA/动态数据存储/开发中心/KodExplorer/admin:/var/www/html/data/User/admin/home \
> -v /share/CACHEDEV3_DATA/动态数据存储/开发中心/KodExplorer/fiona:/var/www/html/data/User/fiona/home \
> -v /share/CACHEDEV3_DATA/动态数据存储/开发中心/KodExplorer/default:/var/www/html/data/User/default/home \
> --restart always \
> baiyuetribe/kodexplorer
> ```
>
> 映射目录中，`fiona/home`目录是用户`fiona`所配置的用户文件夹，将其映射事先映射到NAS物理机的HDD硬盘的某个路径文件夹，以此能够在kodexplorer中使用用户fiona来管理NAS物理机上的文件。
>
> 同时，反过来理解，也可以看作是将kodexplorer中fiona的用户文件持久化地存储在容器外部的NAS物理机中；
>
> 【注意】：
>
> 将容器配置好并安装好之后，容器中的确生成了`/var/www/html/data/User/fiona/home/`文件夹路径，但在登陆kodexplorer，使用admin创建fiona用户后，kodexplorer并不能直接使用该路径会作为fiona的用户文件夹，而是为fiona用户另外创建一个目录（例如：`/var/www/html/data/User/fiona012/home`）。具体原因，猜测可能是权限问题。解决办法如下：
>
> 第一种方法（未实践，理论上可行）。将容器随便配置并安装好后，然后配置好fiona用户，待其创建好用户文件夹后，利用`docker commit`命令将此时的容器生成新的docker镜像。然后删除该容器，重新安装这个新的docker镜像，此时这个新镜像里就已经包含了该用户目录，然后利用`docker run`重新配置好映射路径并安装即可。[Docker容器创建后怎样更改“目录映射”，“端口映射”关系，同时解答现有的网上答案，修改后不生效的问题。](https://blog.csdn.net/qq_25476571/article/details/113744254)
>
> 第二种方法（实践并成功）。
>
> 1. 先按照正常路径映射部署kodexplorer；
> 2. admin登录kodexplorer，修改fiona文件夹名为其他（如fiona01）（这个fiona文件夹因为映射缘故，它在kodexplorer里是无法删除的，只能清空其内文件及子文件夹）；
> 3. 创建fiona用户，kodexplorer会生成fiona用户文件夹；
> 4. 将fiona文件夹里的内容复制到fiona01；
> 5. 删除fiona文件夹，修改fiona01文件夹名字为fiona即可；
> 6. admin账户的用户文件夹映射到物理机的方法同理；
>

【安装步骤】

1. 映射目录改为直接映射`data`文件夹，包含所有用户文件。加入`kingchl-net`网桥，并配置静态`IPv4`：

   正式版本：

   ```bash
   docker run -d -p 9012:80 --name 'kodexplorer' \
   -v /share/CACHEDEV3_DATA/DYNAMIC/ARCHIVE/Docker/KodExplorer/data:/var/www/html/data \
   -e PUID=1000 \
   -e PGID=1000 \
   -e TZ=Asia/Shanghai \
   --net 'kingchl-net' \
   --ip '172.18.0.20' \
   --restart always \
   baiyuetribe/kodexplorer:latest
   ```

2. 版本为4.4，`admin`第一次进入后根据提醒，更新至最新版本即可；

3. 反向代理`swag`配置文件`kodcloud.subdomain.conf`

   ```nginx
   server {
       # 监听swag容器的443端口，此端口映射宿主机（NAS）的8043的端口，并由路由器9999端口转发；
       listen 443 ssl;
       listen [::]:443 ssl;
   
       server_name kodcloud.kingchl.cn;
   
       include /config/nginx/ssl.conf;
   
       client_max_body_size 0;
   
       location / {
           #include /config/nginx/proxy.conf;
           #include /config/nginx/resolver.conf;
           set $upstream_app 172.18.0.20;
           set $upstream_port 80;
           set $upstream_proto http;
           proxy_pass $upstream_proto://$upstream_app:$upstream_port;
   
       }
   }
   ```

4. 2022/1/9日发现，`kodexplorer`可以在容器内更新版本，不管是否重启容器，只要容器没有重新从其镜像处安装，这些更新都是有效保存的。但是`docker`版的`Nginx`改个`/etc/hosts`文件，重启`Nginx`容器后都会被重置，待研究；难道是因为`kodexplorer`是`docker run`安装，而`Nginx`是`docker-compose`安装？但我重启`Nginx`容器使用的是`docker restart`命令，不是`docker-compose restart`，更不是`docker-compose up`。

链接：

- [简述 - 可道云KODExplorer-OpenAPI及开发文档_企业网盘_企业云盘_私有云_云盘_网盘 (kodcloud.com)](http://doc.kodcloud.com/#/start)

- [KODExplorer: KodExplorer是一款快捷高效的私有云和在线文档管理系统GPL v3开源协议. (gitee.com)](https://gitee.com/kalcaddle/KODExplorer)

- [kalcaddle](https://github.com/kalcaddle)/**[KodExplorer](https://github.com/kalcaddle/KodExplorer)**

#### N）部署宝塔面板

【安装步骤】

1. GitHub：**[baota](https://github.com/pch18-docker/baota)**

   ```bash
   # 拉取镜像（挺大的，5个G）
   docker pull pch18/baota
   # 部署容器
   docker run -it --name baota --privileged=true --shm-size=1g \
   -p 9014:80 -p 9015:443 -p 9016:8888 -p 9017:888 \
   -v /share/CACHEDEV3_DATA/DYNAMIC/ARCHIVE/Docker/baota/wwwroot:/www/wwwroot \
   #-v baota-volume:/www \
   --net 'kingchl-net' \
   --ip '172.18.0.60' \
   --restart always \
   -d pch18/baota
   ```

2. 登陆地址 `http://{{面板ip地址}}:9016`，初始账号 `username`，初始密码：所给的初始密码不对；

   ```bash
   # 命令行进入容器
   docker exec -it baota bash
   # 执行bt命令，进入宝塔面板CLI
   bt
   ===============宝塔面板命令行==================
   (1) 重启面板服务           (8) 改面板端口
   (2) 停止面板服务           (9) 清除面板缓存
   (3) 启动面板服务           (10) 清除登录限制
   (4) 重载面板服务           (11) 取消入口限制
   (5) 修改面板密码           (12) 取消域名绑定限制
   (6) 修改面板用户名         (13) 取消IP访问限制
   (7) 强制修改MySQL密码      (14) 查看面板默认信息
   (22) 显示面板错误日志      (15) 清理系统垃圾
   (23) 关闭BasicAuth认证     (16) 修复面板(检查错误并更新面板文件到最新版)
   (24) 关闭谷歌认证          (17) 设置日志切割是否压缩
   (25) 设置是否保存文件历史副本  (18) 设置是否自动备份面板
   (0) 取消
   ===============================================
   请输入命令编号：
   # 修改密码即可,改为cheng5969
   ```

3. 

【注意事项】

- `/www`文件夹默认保存在`docker volume`卷中， `/www/wwwroot`建议映射到宿主机的目录下，方便上传网站代码等文件。安装完成后以后可以随时使用内置升级,升级到最新版本。由于面板数据都保存在持久化的卷中, 即使删除容器（不删除`volume`）后重新运行，原来的面板和网站数据都能得到保留；

  ```bash
  # 要删除容器及容器关联的volumes
  docker rm -fv baota
  # 查看所有容器卷
  docker volume ls
  # 查看卷详细信息
  docker volume inspect volumeID(or volumeName)
  # 单独删除docker volume
  docker volume rm volumeID(or volumeName)
  ```

- `baota`容器的`volume`建议就用它默认生成的，不要自定义`volume`去挂载；

- **如需停止或者重启宝塔**，应直接在`docker`容器层面操作，不要在宝塔面板内部操作！启动宝塔容器时，将自动启动所有服务。容器版宝塔面模比较脆弱，容易出问题，尽量少改动；

- 不要轻易改变`docker`文件夹及文件的权限。这里就因需要在威联通`kingchl`用户下使用`filestation`查`lib/docker/volumes`，改变文件夹权限，继而发生网站出错的情况。使用`baota`镜像重新安装无法解决，最后使用`QTS`快照还原整个系统盘解决问题；

- 宝塔面板中，虚拟主机与虚拟主机之间`server_name`不能相同，会导致宝塔的nginx将服务会一直指向先建立的站点，直至前面建立的站点被删除或者`server_name`被修改；


- `php7.3`启动不了的解决办法，并多试几次：

  > [An another FPM instance seems to already listen on /tmp/php-cgi-71.sock_温玉兰亭的博客-CSDN博客](https://blog.csdn.net/weixin_36521716/article/details/87374913)
  >
  > 今天在更新配置重启php-fpm服务的时候遇到了这样的一个问题：
  >
  > An another FPM instance seems to already listen on /tmp/php-cgi-71.sock
  >
  > 只要将/tmp/php-cgi-73.sock删除在重新启动php-fpm即可解决

- 记得在`php`管理里面安装`Fileinfo`扩展；

##### a）【部署简单图床】

> 也可以使用国外的一款开源图床，它拥有`docker`版：
>
> ```bash
> docker pull lycheeorg/lychee
> ```
>
> Github：[Lychee-Docker](https://github.com/LycheeOrg/Lychee-Docker)（`docker`版）
>
> Github：[ Lychee](https://github.com/LycheeOrg/Lychee)（网站版）

##### Github：[EasyImages2.0](https://github.com/icret/EasyImages2.0)，安装参考：[简单图床-EasyImage2.0 使用手册](https://www.kancloud.cn/easyimage/easyimage)；

【安装步骤】

1. 准备工作`1`~`5`，配置`swag`反向代理：

2. 进入`DNSPod`控制台，添加`DNS`记录，子域名：`easyimage`，并在`DDNS-go`中添加动态域名解析；

3. 在`swag`的`docker-compose`文件中添加`easyimage`项；

4. 在`swag`中添加`nginx`虚拟主机配置文件：`easyimage.subdomain.conf`，配置子域名反向代理：

   ```nginx
   ## Version 2021/05/18
   # for easyimage
   
   server {
       # 监听swag容器的443端口，此端口映射宿主机（NAS）的8043的端口，并由路由器9999端口转发；
       listen 443 ssl;
       listen [::]:443 ssl;
   
       # 宝塔的nginx会接收此server_name，并且据此将服务传递给虚拟主机：easyimage；
       server_name easyimage.*;
   
       include /config/nginx/ssl.conf;
   
       client_max_body_size 0;
   
       # enable for ldap auth, fill in ldap details in ldap.conf
       #include /config/nginx/ldap.conf;
   
       # enable for Authelia
       #include /config/nginx/authelia-server.conf;
   
       location / {
           # enable the next two lines for http auth
           #auth_basic "Restricted";
           #auth_basic_user_file /config/nginx/.htpasswd;
   
           # enable the next two lines for ldap auth
           #auth_request /auth;
           #error_page 401 =200 /ldaplogin;
   
           # enable for Authelia
           #include /config/nginx/authelia-location.conf;
   
           include /config/nginx/proxy.conf;
           include /config/nginx/resolver.conf;
           set $upstream_app baota;  # kingchl-net网桥下，容器之间可通过容器名相互访问；
           set $upstream_port 80;  # 传递给宝塔容器的80端口，而不是其宿主机映射端口9014；
           set $upstream_proto http;
           proxy_pass $upstream_proto://$upstream_app:$upstream_port;
       }
   }
   ```

5. 完成`swag`配置，重新安装`swag`容器，并利用其他站点测试`swag`是否正常启动；

6. 进入`docker`宝塔面板，建立站点，域名填：`easyimage.kingchl.cn:9999`，该网站不需要数据库，但依然给它分配个数据库（数据库名：`easyimage`，用户名`easyimage`，密码`cheng5969`），`php`版本目前选择`PHP7-3`版本，修改站点根文件夹名为`easyimage`；

7. 先在宝塔中配置供宿主机（`TS264C NAS`）访问的局域网`nginx`配置文件：

   ```nginx
   # 宝塔在保存好nginx配置文件的时候就自己重加载了，不需要nginx -s reload；
   # 配置文件修改完毕后，如浏览器访问不到，尝试清除浏览器cookie；
   # 因为宝塔为docker版，根据容器实际配置，修改服务器名及端口如下
   server
   {
       listen 80;  # 👈👈baota的80端口映射在宿主机（NAS）的9014端口，局域网中使用192.168.50.61:9014访问；
       server_name 192.168.50.61;  # 👈👈easyimage域名暂无法访问，修改为宿主机局域网IP；
       index index.php index.html index.htm default.php default.htm default.html;
       root /www/wwwroot/easyimage;  # 站点文件夹，存放网站源文件
       
       # 👈👈Nginx环境中禁止在上传目录中运行PHP程序，放在root /../easyimage;之后
   	# 上传目录禁止运行`PHP`程序
       location ~* ^/(i|public)/.*\.(php|php5)$
       {
           deny all;
       }
       # 👈👈Nginx环境中禁止在上传目录中运行PHP程序
       location ^~ /config/
       {
           deny all;
       }
       
       #SSL-START SSL相关配置，请勿删除或修改下一行带注释的404规则
       #error_page 404/404.html;
       #SSL-END
       
       #ERROR-PAGE-START  错误页配置，可以注释、删除或修改
       #error_page 404 /404.html;
       #error_page 502 /502.html;
       #ERROR-PAGE-END
       
       #PHP-INFO-START  PHP引用配置，可以注释或修改
       include enable-php-73.conf;
       #PHP-INFO-END
       
       #REWRITE-START URL重写规则引用,修改后将导致面板设置的伪静态规则失效
       include /www/server/panel/vhost/rewrite/easyimage.kingchl.cn.conf;
       #REWRITE-END
       
       #禁止访问的文件或目录
       location ~ ^/(\.user.ini|\.htaccess|\.git|\.svn|\.project|LICENSE|README.md)
       {
           return 404;
       }
       
       #一键申请SSL证书验证目录相关设置
       location ~ \.well-known{
           allow all;
       }
       
       location ~ .*\.(gif|jpg|jpeg|png|bmp|swf)$
       {
           expires      30d;
           error_log /dev/null;
           access_log /dev/null;
       }
       
       location ~ .*\.(js|css)?$
       {
           expires      12h;
           error_log /dev/null;
           access_log /dev/null; 
       }
       
       access_log  /www/wwwlogs/easyimage.kingchl.cn.log;
       error_log  /www/wwwlogs/easyimage.kingchl.cn.error.log;
   }
   ```

8. 访问局域网`URL`：http://192.168.50.61:9014，进行建站测试。如建站成功，页面将显示”恭喜建站成功“的信息（`.html`文件不依赖于`php`）；

9. 新建一个文件夹存放网站源文件安装包，从`Github`下载源文件：

   ```bash
   git clone https://github.com/icret/EasyImages2.0.git
   ```

10. 删除新建站点的文件夹内所有文件；

11. 将`easyimages2.0`文件夹内所有网站源文件移动到站点文件夹根目录（目录级别为：`index.php`出现在站点根目录）；

12. 刷新浏览器（http://192.168.50.61:9014），会显示网站安装环境检测页面（如果第3步正常，这里访问不到极有可能是**`php`挂了**）（先不要修改`conf`文件中的`URL`，`URL`后续将一直在网页中设置）；

    【注意】：此时直接域名+广域网访问，是无法跳转到【EasyIamge 2.0 安装环境检测】网站安装界面的，即使提前在`conf`文件中设置`URL`：https://easyimage.kingchl.cn:9999；

13. 如环境检测结果提示文件夹权限问题，是因为网站源码在下载时使用的是`root`账户，所以须将所有文件（夹）、子文件（夹）所有者配置为`www`账户，权限配置为`755`。

14. 安装`php`扩展程序：`fileinfo`、`imagemagick`，重启`php`；

15. `php7.3`会动不动就挂掉，不知道咋回事。重启`php7.3`后有时仍显示停止状态，但网站已能正常工作；

16. 根据【下一步】提示，暂先设置【网站域名】及【图片链接域名】为局域网`URL`：http://192.168.50.61:9014；

17. 根据【下一步】提示，设置用户名及密码等信息（`admin`，`cheng5969`），填完进入登陆页面；

    > 如果忘记账号可以打开->`/config/config.php`文件->找到user对应的键值->填入
    >
    > 如果忘记密码请将密码->转换成`32`位`MD5`小写->[转换网址](https://md5jiami.bmcx.com/)->打开`/config/config.php`文件->找到`password`对应的键值->填入

18. 登陆时验证码**区分**大小写；

19. 第一次登陆网站会显示更详细的【简单图床-EasyImage2.0安装环境监测】，环境还缺什么就安装什么，如下图：

    <img src="https://easyimage.kingchl.cn:9999/i/2022/01/19/简易图床-EasyImage2.0安装环境检测.png" alt="简单图床 - EasyImage" style="zoom:80%;" />

20. 刷新后进入网站，上传一张图片，并访问生成的图片`URL`，测试网站功能是否正常；

21. 如正常，下面开始配置广域网访问；

22. 进入网站设置页面，将【网站域名】及【图片链接域名】**重新修改**为广域网`URL`：https://easyimage.kingchl.cn:9999，保存。此时刷新网站页面（局域网`URL`：http://192.168.50.61:9014），页面上应该只剩下文字，没有图片，或者干脆无法显示；

23. 参照第`7`步，修改宝塔面板中`easyimage`网站的`nginx`配置文件，适配`server_name`：

    ```nginx
    server
    {
        listen 80;  # baota容器中nginx的监听端口80；
        server_name easyimage.*;  # 👈👈👈，改为外网域名，适配server_name，外网中使用https://easyimage.kingchl.cn:9999访问；
        index index.php index.html index.htm default.php default.htm default.html;
        root /www/wwwroot/easyimage;
        
        # 👈👈Nginx环境中禁止在上传目录中运行PHP程序，放在root /../easyimage;之后
    	# 上传目录禁止运行`PHP`程序
        location ~* ^/(i|public)/.*\.(php|php5)$
        {
            deny all;
        }
        # 👈👈Nginx环境中禁止在上传目录中运行PHP程序
        location ^~ /config/
        {
            deny all;
        }
        
        #SSL-START SSL相关配置，请勿删除或修改下一行带注释的404规则
        #error_page 404/404.html;
        #SSL-END
        
        #ERROR-PAGE-START  错误页配置，可以注释、删除或修改
        #error_page 404 /404.html;
        #error_page 502 /502.html;
        #ERROR-PAGE-END
        
        #PHP-INFO-START  PHP引用配置，可以注释或修改
        include enable-php-73.conf;
        #PHP-INFO-END
        
        #REWRITE-START URL重写规则引用,修改后将导致面板设置的伪静态规则失效
        include /www/server/panel/vhost/rewrite/easyimage.kingchl.cn.conf;
        #REWRITE-END
        
        #禁止访问的文件或目录
        location ~ ^/(\.user.ini|\.htaccess|\.git|\.svn|\.project|LICENSE|README.md)
        {
            return 404;
        }
        
        #一键申请SSL证书验证目录相关设置
        location ~ \.well-known{
            allow all;
        }
        
        location ~ .*\.(gif|jpg|jpeg|png|bmp|swf)$
        {
            expires      30d;
            error_log /dev/null;
            access_log /dev/null;
        }
        
        location ~ .*\.(js|css)?$
        {
            expires      12h;
            error_log /dev/null;
            access_log /dev/null; 
        }
        
        access_log  /www/wwwlogs/easyimage.kingchl.cn.log;
        error_log  /www/wwwlogs/easyimage.kingchl.cn.error.log;
    }
    ```

    保存配置文件。

24. 浏览器访问广域网`URL`：https://easyimage.kingchl.cn:9999，此时文字、图片又都回来了，并且网站能够正常访问；

25. 至此建站成功，可以正常使用简单图床；

26. 外链测试：

    ![简单图床 - EasyImage](https://easyimage.kingchl.cn:9999/i/2022/01/18/zdo5wn.jpg)

【安装`PicGo`客户端】

`PicGo`是一个开源图床客户端，支持`Github`图床、阿里云`OSS`等，同时也支持自定义`Web`图床，`Github`地址：[PicGo](https://github.com/Molunerfinn/PicGo)；

> ##### 1. 下载最新版[PicGo-windows版](https://www.545141.com/usr/themes/armx/ext/link/?url=https://github.com/Molunerfinn/PicGo/releases)或者[PicGo-mac版](https://www.545141.com/usr/themes/armx/ext/link/?url=https://github.com/Molunerfinn/PicGo/releases)（我下载的版本是：2.3.1-beta.2）
>
> ##### 2. 安装后在插件设置中搜索web-uploader并安装（下载插件可能需要[node.js](https://www.545141.com/usr/themes/armx/ext/link/?url=https://nodejs.org/zh-cn/)插件）
>
> ##### 3. 图床设置-自定义Web图床中按照如下方式填写，然后点击确定并设置为默认图床。

1. 客户端支持多平台，`windows`客户端采用[PicGo-Setup-2.3.1-beta.2-x64.exe](https://github.com/Molunerfinn/PicGo/releases/download/v2.3.1-beta.2/PicGo-Setup-2.3.1-beta.2-x64.exe)版本；

2. 安装后在插件设置中搜索`web-uploader`并安装（下载插件可能需要[Node.js](https://nodejs.org/zh-cn/)插件），我这里选择的插件版本为：`web-uploader-custom-url-prefix 1.1.1`自定义web图床（增加自定义图片`URL`前缀）；

3. 进入easyimage网站设置页，在上传设置里打开【开启API上传】；

4. 【图床设置】👉【自定义Web图床】中按照如下方式填写，然后点击确定并设置为【默认图床】：

   ```yaml
   # 输入你的网站api地址
   API地址: https://easyimage.kingchl.cn:9999/api/index.php
   POST参数名: image
   JSON路径: url
   # 这里输入你网站生成的token
   自定义Body: {"token":"10fea62d1e00daefb2dd2e8244be339c"}
   # 自定义图片URL，先随便设置个，后面重新再配置为空白，这样就不会出现图片链接为nullhttps://..的情况
   ```

【安装`chrome`插件】

1. 作者`Blog`处下载`EasyImage-Chrome.crx`，地址：[www.545141.com](https://www.545141.com/)
2. 不受信任的插件可通过更改后缀名位`rar`，解压缩后得到插件文件夹；
3. 浏览器扩展管理页面里找到【安装已解压插件】，即可；

【注意事项】

- 图片库路径：`/i`，备份时保存此文件夹；

- 安全配置
  
  > - Apache环境在上传目录添加配置文件`.htaccess` 使上传目录不可运行PHP程序（默认存在)
  >
  > ```
  > <FilesMatch "\.(?i:php|php3|php4|php5)">
  > Order allow,deny
  > Deny from all
  > </FilesMatch>
  > ```
  >
  > - Nginx环境限制上传目录禁止运行`PHP`程序：
  >
  > ```
  >     # 禁止运行php的目录 "i"是你的上传图片目录
  >     location ~ /(i)/.*.(php|php5)?$ {
  >         deny all;
  >     }
  > ```
  >
  > - 或者参考：https://www.545141.com/981.html

  禁止浏览器访问网站源文件中的相关重要文件夹：
  
  ```nginx
  # 禁止从浏览器访问网站源文件中config文件夹
      location ^~ /config/
      {
          deny all;
      }
  # 还有Public文件夹等
  ```
  
- 删除软件作者提供的`token`，因为它们已经不安全；

  ```php
  /* 路径：easyimage/config/api_key.php */
  $tokenList = array(
      0 => '10fea62d1e00daefb2dd2e8244be339c',
      
    /*0 => '8337effca0ddfcd9c5899f3509b23657',
      1 => '1c17b11693cb5ec63859b091c5b9c1b2',*/	
  );
  ```

  

- ![](https://easyimage.kingchl.cn:9999/i/2022/01/19/海贼_0.jpg)

##### b）部署DokuWiki

【注意事项】

- 安全配置

  > 参考：[基于Nginx的Dokuwiki敏感目录访问限制]([基于Nginx的Dokuwiki敏感目录访问限制 | 知行近思 (annhe.net)](https://www.annhe.net/article-2415.html))
  >
  > ```nginx
  > # Nginx虚拟主机中添加如下locaion，禁止从网站访问网站源码中的这些文件夹；
  > location ^~ /conf/
  >     {
  >         deny all;
  >     }
  >     location ^~ /data/
  >     {
  >         deny all;
  >     }
  >     location ^~ /inc/
  >     {
  >         deny all;
  >     }
  >     location ^~ /bin/
  >     {
  >         deny all;
  >     }
  > ```
  >
  > 或采用dokuwiki官方关于[Nginx访问限制的配置](https://www.dokuwiki.org/security#deny_directory_access_in_nginx)
  >
  > ```nginx
  > location ~ /(data|conf|bin|inc)/ {
  >       deny all;
  >     }
  > ```
  >
  > 

- 部署完进入主页正常，但是在登陆后，页面重定向时会丢失端口号`9999`。因为dokuwiki自动检测并重定向给Client的URL为https://dokuwiki.kingchl.cn:80。

  解决办法有两个：

  - 1、进入【管理】👉【配置管理器】👉【`baseurl`】，填入https://dokuwiki.kingchl.cn:9999；

  - 2、直接在网站文件`/conf/local.php`中添加`$conf['baseurl'] = 'https://dokuwiki.kingchl.cn:9999';`；

  - 【注意】：不管那种方法，`URL`的最后（即`9999`后面）不要加`/`；

  - 【注意】：`/conf/dokuwiki.php`为默认配置项，当`dokuwiki`的某项配置参数被修改时，这些新修改的配置会记录在`/conf/local.php`，并覆盖生效；

  > 参考地址：https://www.dokuwiki.org/config:baseurl
  >
  > # Configuration Setting: `baseurl`
  >
  > URL to server including protocol - blank for autodetect
  >
  > - Type: String
  > - Default:``
  >
  > The path you should set here, is the path of server root. Eg. if your wiki is available at `http://www.yourserver.com:port/dokuwiki/` , you should set `baseurl` to `http://www.yourserver.com:port`, **ending without slash**.
  >
  > *Note: The `:port` portion is relevant only if the web server is running on a non-standard port (i.e. not port 80). You can omit that portion if you are running on the standard port.*

- 

#### O）部署`Organizr`

导航页，类似`Heimdall`。`Github`：[causefx](https://github.com/causefx)/**[Organizr](https://github.com/causefx/Organizr)**，官网：[Organizr – Forget Bookmarks](https://organizr.app/)

【安装步骤】

1. `docker run`安装

   ```bash
   docker run -d \
     --name=organizr \
     -v /share/CACHEDEV1_DATA/Public/Container/Docker/Organizr/config:/config \
     --net 'kingchl-net' \
     --ip '172.18.0.6' \
     --restart always \
     -e PGID=1000 -e PUID=1000 \
     -p 9020:80 \
     -e fpm="false" #optional \
     -e branch="v2-master" #optional \
     organizr/organizr
   ```

2. `organizr`账户信息:

   kingchl	cheng5969	port:9020(80)	HASH密码：cheng5969，注册密码：cheng5969

3. 

#### P）部署`OnlyOffice`

GitHub：[Docker-DocumentServer](https://github.com/ONLYOFFICE/Docker-DocumentServer)

【安装步骤】

1. 编辑`docker run`

   ```bash
   docker run -it --name=oods -p 9018:80 \
     -v /share/CACHEDEV3_DATA/DYNAMIC/ARCHIVE/Docker/onlyofficeDS/logs:/var/log/onlyoffice  \
     -v /share/CACHEDEV3_DATA/DYNAMIC/ARCHIVE/Docker/onlyofficeDS/data:/var/www/onlyoffice/Data  \
     -v /share/CACHEDEV3_DATA/DYNAMIC/ARCHIVE/Docker/onlyofficeDS/lib:/var/lib/onlyoffice \
     -v /share/CACHEDEV3_DATA/DYNAMIC/ARCHIVE/Docker/onlyofficeDS/rabbitmq:/var/lib/rabbitmq \
     -v /share/CACHEDEV3_DATA/DYNAMIC/ARCHIVE/Docker/onlyofficeDS/redis:/var/lib/redis \
     -v /share/CACHEDEV3_DATA/DYNAMIC/ARCHIVE/Docker/onlyofficeDS/db:/var/lib/postgresql \
     -e PUID=1000 -e PGID=1000 -e TZ=Asia/Shanghai \
     --net 'kingchl-net' \
     --ip '172.18.0.25' \
     --restart always \
     -d onlyoffice/documentserver:latest
     
   ```

2. 在`OnlyOffice`配置文件中添加自动保存设置，命令行进入`OnlyOffice`容器内部：`/etc/onlyoffice/documentserver/local.json`

   ```json
   {
     "services": {
       "CoAuthoring": {
         "autoAssembly": {
                    "enable": true,
                    "interval": "5m"
         },
         "sql": {
           "type": "postgres",
           "dbHost": "localhost",
           "dbPort": "5432",
           "dbName": "onlyoffice",
           "dbUser": "onlyoffice",
           "dbPass": "onlyoffice"
         },
         "token": {
           "enable": {
             "request": {
               "inbox": false,
               "outbox": false
             },
             "browser": false
           },
           "inbox": {
             "header": "Authorization"
           },
           "outbox": {
             "header": "Authorization"
           }
         },
         "secret": {
           "inbox": {
             "string": "secret"
           },
           "outbox": {
             "string": "secret"
           },
           "session": {
             "string": "secret"
           }
         }
       }
     },
     "rabbitmq": {
       "url": "amqp://guest:guest@localhost"
     }
   }
   ```

   重启`OnlyOffice`: 

   ```bash
   # 容器内部：
   supervisorctl restart all
   ```

3. 【注意】：当`Seafile`使用`HTTPS`协议时，`Onlyoffice`也必须能使用`HTTPS`访问时，才可被`Seafile`集成调用；

   配置`Nginx`反向代理，同时申请`SSL`证书。`swag`配置文件`oods.subdomain.conf`，参考地址：[Using ONLYOFFICE Docs behind the proxy](https://helpcenter.onlyoffice.com/en/installation/docs-community-proxy.aspx) 

   ![onlyoffice for swag](https://easyimage.kingchl.cn:9999/i/2022/02/19/onlyoffice-for-swag_0.png)

   👉 [document-server-proxy](https://github.com/ONLYOFFICE/document-server-proxy)

   ```nginx
   ## Version 2021/05/18
   # Make sure that your dns has a cname set for onlyoffice named "documentserver"
   # Make sure that the onlyoffice documentserver container is named "documentserver"
   
   server {
       listen 443 ssl;
       listen [::]:443 ssl;
   
       server_name oods.*;
   
       include /config/nginx/ssl.conf;
   
       client_max_body_size 0;
   
       #enable for ldap auth, fill in ldap details in ldap.conf
       #include /config/nginx/ldap.conf;
   
       # enable for Authelia
       #include /config/nginx/authelia-server.conf;
   	
   	# 添加。20220219
   	add_header X-Content-Type-Options nosniff;
   
       location / {
           #enable the next two lines for http auth
           #auth_basic "Restricted";
           #auth_basic_user_file /config/nginx/.htpasswd;
   
           #enable the next two lines for ldap auth
           #auth_request /auth;
           #error_page 401 =200 /ldaplogin;
   
           # enable for Authelia
           #include /config/nginx/authelia-location.conf;
   
           #include /config/nginx/proxy.conf;
   		
   		# 添加。20220219
   		proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection $proxy_connection;
           proxy_set_header X-Forwarded-Host $the_host;
           proxy_set_header X-Forwarded-Proto $the_scheme;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
   		
           include /config/nginx/resolver.conf;
           set $upstream_app oods;
           set $upstream_port 80;
           set $upstream_proto http;
           proxy_pass $upstream_proto://$upstream_app:$upstream_port;
   		
   		# 添加。20220219
   		proxy_http_version 1.1;
   
       }
   }
   ```

   还需在`swag`的`nginx.conf`的`HTTP`模块中添加如下代码：

   ```nginx
   	# 以下为onlyoffice添加。20220219
   	map $http_host $this_host {
   		"" $host;
   		default $http_host;
   	}
   
   	map $http_x_forwarded_proto $the_scheme {
   		default $http_x_forwarded_proto;
   		"" $scheme;
   	}
   
   	map $http_x_forwarded_host $the_host {
   		default $http_x_forwarded_host;
   		"" $this_host;
   	}
   
   	map $http_upgrade $proxy_connection {
   		default upgrade;
   		"" close;
   	}
   	# 以上为onlyoffice添加。20220219
   ```

2. 



#### X）`typecho`导航主题`Webstack`

地址：[typecho 导航主题Webstack 钻芒博客二开美化版](https://www.zmki.cn/5366.html)；

`QQ`群：`737656800`，`991742215`；

NAS中收藏版本：`V1.11.1(带排序)_WebStack_钻芒二开.zip`

#### Y）Code-Server



#### Z）部署`Ubuntu`

```bash
## 此处部署为最小系统，用于自己制作容器，这里不分配其宿主机端口及kingchl-net的静态IP；
# 拉取镜像
docker pull ubuntu/nginx:1.18-20.04_beta
# 安装容器，因是最小系统，暂不分配端口映射、区域及网络
docker run -itd --name base_ubuntu ubuntu bash
# 登陆该最小系统
docker exec -it base_ubuntu bash
# 更新apt源
apt-get update -y && apt-get upgrade -y
# 接下来先安装 tzdata 库，选择时区，再去安装其他的 ，过程中根据提示选择【6 亚洲】，【70 上海】
apt-get install -y tzdata && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
# 接下来安装常用包 ，先安装ifconfig命令。
apt install net-tools
# 接下来再安装vim
apt install vim
# 再安装ssh服务，必备
apt install openssh-server
# 安装结束之后看看服务是否启动
ps aux | grep ssh
# 修改配置文件，允许远程连接
vim /etc/ssh/sshd_config
# 修改后
LoginGraceTime 120
PermitRootLogin yes
StrictModes yes
# 重启服务
service restart ssh
```

现在常用的服务都装完了，还装其他的就看每个人需求；

> 配置文件中文乱码
>
> https://www.cnblogs.com/beile/p/12910166.html
>
> 解压文件中文乱码
>
> https://www.cnblogs.com/beile/p/13032148.html

制作完成后，docker commit生成新的镜像保存；

## 七、反向代理与域名

### 1、反向代理与安全访问

安全地访问家庭服务器是我们的【目的】，以往都是直接将服务器暴露，

反向代理的【优点】：

- 能够将各类服务器隐藏在反向代理服务器之后，隐藏这些服务的端口，使它们不暴露在互联网上；

- 同时又能够将不同的服务分配到不同的子域名（二级域名），访问时不需要再去记忆各种端口号，方便又好记。

`nginx`

通常访问的`URL`地址，如http://172.18.0.60:80/，这最后的`/`同`location`模块的`/`。代表的是访问到的根目录，它是一个相对路径，表示`nginx`配置文件`nginx.conf`的父文件夹`conf`所在的目录；

```nginx
server {
	listen 80;
    server_name localhost;	#172.18.0.60
    
    location / {
        # 这里的html文件夹与conf文件夹同目录，在nginx/ ；
        root html;
        index index.html index.htm;
    }
}# kkk
```



修改完`nginx.conf`文件须及时重加载：

```bash
# 在nginx主程序目录时
./nginx -s reload
# 若nginx已启动，也可以直接：
nginx -s reload
```



`Nginx`的进程模型分为主进程`master`以及`worker`

```bash
# 查看Nginx进程
ps -ef|grep nginx
# 可看出master进程由root用户控制，worker进程则由www用户（由nginx.conf配置）控制；
root       822     1  0 Jan16 ?        00:00:00 nginx: master process /www/server/nginx/sbin/nginx -c /www/server/nginx/conf/nginx.conf
www       4285   822  0 10:06 ?        00:00:00 nginx: worker process
www       4286   822  0 10:06 ?        00:00:00 nginx: worker process
www       4287   822  0 10:06 ?        00:00:00 nginx: worker process
www       4288   822  0 10:06 ?        00:00:00 nginx: worker process
www       4289   822  0 10:06 ?        00:00:00 nginx: cache manager process
```

`master`通常只有一个，`worker`可配置很多个，配置位置在`nginx/conf/nginx.conf`中的`worker_processes`项。注意，`worker_connections`表示连接数，两者是有区别的；

`master`负责监听端口、处理管理员操作及`nginx`系统管理；

`worker`对争抢到的`client`进行读取👉解析👉处理👉响应，将`clinet`的请求响应给`client`；

`nginx`事件处理：在`linux`系统中可采用异步阻塞（`linux`系统的`use epoll;`），单`worker`进程可处理6~8万个`connections`请求；

`http`模块：

`server`模块：虚拟主机配置，可以有多个；

`location`模块：路由规则，表达式；

`upstram`模块：集群，内网服务器，负载均衡；

虚拟主机可以是一个`IP`地址上的不同监听端口所对应的`server`，也可以是不同`IP`地址（如容器之间）的相同端口（比如常见的80端口）对应的`server`；

`Nginx`常用命令：

```bash
# 查看当前nginx版本号
nginx -v
# 验证配置是否正确
nginx -t
# 重新加载Nginx配置文件，reload命令在swag中无效
nginx -s restart
# 当没有用户连接时退出nginx（仅http请求）
nginx -s quit
# 直接停止nginx
nginx -s stop
```

### 2、域名

希望通过以`https://`这种加密方式对家庭服务器进行外网访问，同时又避免记忆各种端口号

【注意事项】：因`80`端口被封，所以需要指定另外的端口，我这里选择:`9999`，即https://abc.kingchl.top:9999；

我的域名：

| `Aliyun`                | `Dnspod`               |
| ----------------------- | ---------------------- |
| `kingchl.top`（至2032） | `kingchl.cn`（至2025） |

`Qnap Web`管理页使用https://kingchl.top:5001访问，直接在`QTS`控制台里添加服务器`SSL`证书，不经过反向代理；

### 3、`SSL`证书

`CA`在为一个域名（不管是一级域名还是二级域名）颁发ssl证书前，会针对域名所属人进行一项`CAA`验证，通过后`CA`才会下发证书。`CAA`验证有两种方式，

- 服务器文件验证：假设要为一个服务器绑定域名[blog.kingchl.top]()，并为这个域名添加`SSL`证书。事先在该服务器根目录放置`CA`（如`Let's Encrypt`）生成的一个文本文件，然后`CA`通过[blog.kingchl.top]()访问该服务器，读取并校验该服务器下的这个文本文件，证明确实是该域名的所属本人开展的`SSL`申请操作，从而完成`SSL`证书的申请验证；

- 域名解析验证：在域名所归属的域名注册商的`DNS`解析中添加由CA指定的`TXT`类型的主机记录及记录值，`CA`对域名的这条主机记录（实际上也是一个域名，如[_dnsblog.kingchl.top]()）进行访问，验证注册商对这条主机记录的解析值是否正确，从而达到验证所属人的目的，完成`SSL`证书申请验证；

在以验证服务器文件的方式进行验证时，`CA`默认通过`80`端口（[blog.kingchl.top]()，`:80`省略不写）访问该服务器，而大陆家庭宽带`80`端口被运营商封闭，`CA`永远访问不到该服务器，因此无法读取到这个文本文件，验证失效；而作为域名所有者，我正好又有这个域名的`DNS`解析配置权，因此这里采用`DNS`验证的方式申请`SSL`证书；

`Docker`版`Let's Encrypt`：似乎内置了`nginx`

### 4、动态`DNS`

它本质就是个自动化程序，定期地跑到服务器（我这里使用反向代理服务器作为互联网访问的落脚点）所在的端口处查看一下该服务器所在的公网`IP`地址，将其上传给`DNS`解析商（域名注册商，比如阿里云），最终将该公网`IP`同步给这个服务器所在的域名；

### 5、实现方案

#### 实现动态`dns`

> 参考资料：
>
> [ddns-updater](https://github.com/qdm12/ddns-updater)

> 参考资料：
>
> ```bash
> docker run -d \
> -v /share/CACHEDEV1_DATA/Public/Container/Docker/DDNS/config.json:/config.json \
> --network host \
> newfuture/ddns
> ```
>
> [DDNS](https://github.com/NewFuture/DDNS)（[【折腾必懂·实践篇】第三期 域名和花式DDNS实践_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1z7411L7Em?from=search&seid=10853455252256110183&spm_id_from=333.337.0.0)）

【安装步骤】

1. 进入域名解析控制台，这里选择`DNSPod`；

2. 添加`A`记录，填写子域名，填写`IPv4`地址，地址随便填，后面会被DDNS程序更新为真实公网`IP`；

3. 值得注意的是，不同的子域名在`DNS`解析后均为同一个公网`IP`地址，这是没毛病的，这些域名（加端口号）将全都指向`Nginx`反向代理服务器，由`Nginx`负责分发；

4. 在`TS-264C`上安装`jeessy/ddns-go`，实现`DNSPod`上的动态解析（`Aliyun`也可以），`ddns`软件地址：[ddns-go](https://github.com/jeessy2/ddns-go)，它在`docker-hub`上也能找到；

   ```bash
   docker run -d \
   --name ddns-dnspod \
   --restart=always \
   -p 8998:9876 \
   -v /share/CACHEDEV1_DATA/Public/Container/Docker/ddns-go_kingchl.cn:/root \
   --net 'kingchl-net' \
   --ip '172.18.0.4' \
   jeessy/ddns-go
   ```

   <!--`ddns-go`版本号：v3.4.1-->

5. 访问http://localhost:8998访问`ddns-go`的`Web`管理页面，根据提示选择`DNS`解析商，`ID/Token`，`Domains`（一行一个完整的域名，如`seafile.kingchl.cn`，多个域名之间只需回车隔开，不需要任何间隔符）。

   日志文件在`Web`管理页面就能看到，当然在`/root`下也能查看，为隐藏文件；

【注意事项】

​    `DNSPod`上的`API`接口现在升级到了`API3.0`协议（默认），使用的是`SecretId/SecretKey`，而`ddns`需要的`ID/Token`是在`API2.0`协议下的，目前`DNSPod`仍然支持`DNSPod Token`（2022/1/7）

​    本次使用的`DNSPod`的`ID/Token`为：

| ID       | Token                              |
| -------- | ---------------------------------- |
| `283449` | `1f1cdec59a066c33645c3c2823552a9b` |

---

【ddns-go for `kingchl.top`】

```bash
docker run -d \
--name ddns-aliyun \
--restart=always \
-p 8996:9876 \
-v /share/CACHEDEV1_DATA/Public/Container/Docker/ddns-go_kingchl.top:/root \
--net 'kingchl-net' \
--ip '172.18.0.3' \
jeessy/ddns-go
```



---

#### 实现反向代理及SSL证书

【注意】

swag反向代理宿主机的qnap nas web页面时，不仅代理不了（只能显示主页，控制台图标显示不全，文字错位，卡顿，过一会提示服务器断开连接），还出现导致swag停止响应的情况。不执行此代理后要过好一会swag才能恢复，估计跟nginx 缓存有关

*貌似重新配置一边qts.subdomain.conf（将location中https改为http，并注释掉两行include），然后重启swag容器后，nginx就可以恢复正常；

*不要轻易尝试用swag反向代理qnap nas web，已将qts.subdomain.conf配置文件删除。。



采用`linuxserver/swag`（原`linuxserver/letsencrypt`），它是一个融合`nginx`反向代理及`ssl`证书的这里使用`docker-compose`安装，项目地址：**[docker-swag](https://github.com/linuxserver/docker-swag)**。

下载官方`yml`并对应修改，加入`kingchl-net`：

```yaml
---
version: "2.1"
services:
  swag:
    image: lscr.io/linuxserver/swag
    container_name: swag
    cap_add:
      - NET_ADMIN
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Shanghai
      - URL=kingchl.cn
      - VALIDATION=dns
      - SUBDOMAINS=seafile,kodcloud,bookstack,heimdall,wiznote,easyimage,dokuwiki #optional
      - CERTPROVIDER= #optional
      - DNSPLUGIN=dnspod #optional
      - PROPAGATION= #optional
      - DUCKDNSTOKEN= #optional
      - EMAIL=345290729@qq.com #optional
      - ONLY_SUBDOMAINS=false #optional
      - EXTRA_DOMAINS= #optional
      - STAGING=false #optional
    volumes:
      - /share/CACHEDEV1_DATA/Public/Container/Docker/swag/config:/config
    ports:
      - 8443:443
      - 8080:80 #optional
    restart: always
    networks:
      kingchl-net:
        ipv4_address: 172.18.0.2

networks:
  kingchl-net:
    external: true
```

<!--swag版本号：21.12.21-->

【安装步骤】：

1. 创建`docker`自定义`subnet`网段（子网掩码），指定网关，并指定自定义网桥名称`swag-net`：

   ```bash
   docker network create --driver bridge --subnet 172.18.0.0/16 --gateway 172.18.0.1 kingchl-net
   ```

   【注意】掩码为16位；

2. 预设静态路由表：

   | 子域名      | 宿主机端口     | 虚拟主机端口       | 虚拟主机IP   | 站点名      | 容器名          |
   | ----------- | -------------- | ------------------ | ------------ | ----------- | --------------- |
   | -           |                |                    | `172.18.0.1` | 虚拟网关    |                 |
   | `swag`      | `8080`, `8443` | `80`,  `443`       | `..0.2`      | 同右        | `swag`          |
   | -           | `8996`         | `9876(仅内网)`     | `..0.3`      | 同右        | `ddns-aliyun`   |
   | -           | `8998`         | `9876(仅内网)`     | `..0.4`      | 同右        | `ddns-dnspod`   |
   | `heimdall`  | `9002`,`9003`  | `80`, `443(反代)`  | `..0.5`      | 同右        | `Heimdall`      |
   | organizr    | 9020, 9021     | 80                 | ..0.6        |             | organizr        |
   | `seafile`   | `9010`         | `80(反代)`         | `..0.10`     | 同右        | `seafile`       |
   | -           | -              | -                  | `..0.11`     | 同右        | `seafile-db`    |
   | -           | -              | -                  | `..0.12`     | 同右        | `seafile-mcach` |
   | -           | -              | -                  | `..0.13`     | 同右        | `seafile-es`    |
   | `kodcloud`  | `9012`         | `80(反代)`         | `..0.20`     | 同右        | `kodexplorer`   |
   | `oods`      | `9018`, `9019` | `80(反代)`, `443`  | `..0.25`     | 同右        | `oods`          |
   | `bookstack` | `9008`         | `80(反代)`         | `..0.30`     | 同右        | `bookstack`     |
   | -           |                |                    | `..0.31`     | 同右        | `bookstack_db`  |
   | `wiznote`   | `9006`, `9007` | `80(反代)`, `9269` | `..0.40`     | 同右        | `wiz`           |
   | -           | `9004`         | `8083(反代)`       | `..0.45`     | 同右        | `calibre-web`   |
   | -           |                |                    | `..0.50`     |             | `ubuntu`        |
   | `easyimage` | `9014`, `9015` | `80(反代)`, `443`  | `..0.60`     | `easyimage` | `baota`         |
   | -           | `9017`         | `888(仅内网)`      | `..0.60`     |             | `baota`         |
   | -           | `9016`         | `8888(仅内网)`     | `..0.60`     | 宝塔面板    | `baota`         |
   | `dokuwiki`  | `9014`, `9015` | `80(反代)`, `443`  | `..0.60`     | `dokuwiki`  | `baota`         |

   

1. 配置`YML`文件相关参数，如填写需要申请SSH证书的子域名名称等；

2. 初始安装时采用**前台**安装，以便观察`SSL`证书申请情况：

   ```bash
   docker-compose up
   ```

3. 观察安装没有问题，`Ctrl+c`终止该安装进程，停止该容器（如后台安装，则`docker stop swag`）；

4. 进入容器挂载目录：

   ```bash
   # 进入挂载目录（配置申请SSL证书时的域名DNS解析验证）
   cd /share/CACHEDEV1_DATA/Public/Container/Docker/swag/config/dns-conf/
   # 选择DNS验证Provider，这里选择DNSPod，打开并编辑配置文件
   vim dnspod.ini
   ```

7. 如下填入`DNSPod`的`API`信息：

   ```ini
   dns_dnspod_email = "345290729@qq.com"
   dns_dnspod_api_token = "283449,1f1cdec59a066c33645c3c2823552a9b"
   ```

   | ID       | Token                              |
   | -------- | ---------------------------------- |
   | `283449` | `1f1cdec59a066c33645c3c2823552a9b` |

8. 重新启动，重新加载`dnspod.ini`配置文件：

   ```bash
   docker start swag
   ```

   同时查看`swag`启动日志：

   ```bash
   docker logs -f swag
   ```

   命令行进入`swag`容器后，`Nginx`常用命令：

   ```bash
   # 查看当前nginx版本号
   nginx -v
   # 验证配置是否正确
   nginx -t
   # 重新加载Nginx配置文件，reload命令在swag中无效
   nginx -s restart
   ```

   > 【`Nginx`中的`bug`】
   >
   > - `Nginx`服务启动后，执行 [nginx](https://so.csdn.net/so/search?q=nginx) -t 是OK的，但执行 `nginx -s reload` 等命令时报错；
   >
   >   ```bash
   >   nginx: [error] invalid PID number "" in "/run/nginx.pid"
   >   ```
   >
   > 【解决方法】
   >
   > 1. 根据错误提示找到`nginx.pid`位置；
   >
   > 2. 查资料知`nginx.pid`为记录`nginx`服务进程值的文件；
   >
   >    ```bash
   >    # 该文件路径
   >    /run/nginx/nginx.pid
   >    # 查看其大小，并打印该文件
   >    du -h --max-depth=1 ./*
   >    0       ./nginx.pid
   >    # 打印此文件
   >    cat nginx.pid
   >    # 得出nginx.pid为空文件
   >    ```
   >
   > 3. 所以，须将进程值写进`nginx.pid`文件
   >
   >    ```bash
   >    # 查看nginx.config位置
   >    nginx -t
   >    # 刷写一下nginx配置文件
   >    nginx -c /etc/nginx/nginx.conf
   >    # 再次打印nginx.pid文件
   >    cat nginx.pid
   >    5798
   >    # 尝试执行nginx -s命令
   >    nginx -s reload
   >    # 没有报错，即成功
   >    ```
   >
   > 4. 如果以上没用，则考虑`kill`掉`nginx`里面的进程；

9. 测试`HTTPS`访问`Nginx`地址，能带🔒访问到`Nginx`代表创建`swag`成功，并且成功申请到`SSL`证书；

   访问地址：https://subdomain.kingchl.cn:port@Nginx

10. 配置`Nginx`反向代理配置文件

11. 停止`swag`容器，修改`nginx.conf`文件

    因运营商封闭了家庭宽带的默认`web`端口（路由器`80`端口、路由器`443`端口），这里使用路由器`9998`端口代替`80`，`9999`端口代替`443`；

    所以须在`nginx.conf`中，对【`http`重定向到`https`】的`server`模块进行修改，具体如下：

    ```nginx
    # swag/config/nginx/site-confs/default（映射路径）
    # $host → $host:9999
    
    # redirect all traffic to https
    server {
        listen 80 default_server;
        listen [::]:80 default_server;
        server_name _;
        return 301 https://$host:9999$request_uri;
    }
    ```

    后续在浏览器地址栏输入<u>(http://)subdomain.kingchl.cn:9998</u>访问时可直接跳转到：

    URL ： https://subdomain.kingchl.cn:9999

12. 配置子域名反向代理配置文件`*.subdomain.conf`。【注意】文件名中的`*`字段，要与该`conf`配置文件里面的`server_name`保持一致；

13. 截至2022/2/19，`swag`的`nginx.conf`配置文件如下：

    ```nginx
    ## Version 2021/04/27 - Changelog: https://github.com/linuxserver/docker-swag/commits/master/root/defaults/nginx.conf
    
    user abc;
    
    # Set number of worker processes automatically based on number of CPU cores.
    include /config/nginx/worker_processes.conf;
    
    # Enables the use of JIT for regular expressions to speed-up their processing.
    pcre_jit on;
    
    # Configures default error logger.
    error_log /config/log/nginx/error.log;
    
    # Includes files with directives to load dynamic modules.
    include /etc/nginx/modules/*.conf;
    
    events {
        # The maximum number of simultaneous connections that can be opened by
        # a worker process.
        worker_connections 1024;
        # multi_accept on;
    }
    
    http {
        # Includes mapping of file name extensions to MIME types of responses
        # and defines the default type.
        include /etc/nginx/mime.types;
        default_type application/octet-stream;
    
        # Name servers used to resolve names of upstream servers into addresses.
        # It's also needed when using tcpsocket and udpsocket in Lua modules.
        #resolver 1.1.1.1 1.0.0.1 2606:4700:4700::1111 2606:4700:4700::1001;
        include /config/nginx/resolver.conf;
    
        # Don't tell nginx version to the clients. Default is 'on'.
        server_tokens off;
    
        # Specifies the maximum accepted body size of a client request, as
        # indicated by the request header Content-Length. If the stated content
        # length is greater than this size, then the client receives the HTTP
        # error code 413. Set to 0 to disable. Default is '1m'.
        client_max_body_size 0;
    
        # Sendfile copies data between one FD and other from within the kernel,
        # which is more efficient than read() + write(). Default is off.
        sendfile on;
    
        # Causes nginx to attempt to send its HTTP response head in one packet,
        # instead of using partial frames. Default is 'off'.
        tcp_nopush on;
    
        # 以下为onlyoffice添加。20220219
    	map $http_host $this_host {
    		"" $host;
    		default $http_host;
    	}
    
    	map $http_x_forwarded_proto $the_scheme {
    		default $http_x_forwarded_proto;
    		"" $scheme;
    	}
    
    	map $http_x_forwarded_host $the_host {
    		default $http_x_forwarded_host;
    		"" $this_host;
    	}
    
    	map $http_upgrade $proxy_connection {
    		default upgrade;
    		"" close;
    	}
    	# 以上为onlyoffice添加。20220219
    
        # Helper variable for proxying websockets.
        map $http_upgrade $connection_upgrade {
            default upgrade;
            '' close;
        }
    
        # Sets the path, format, and configuration for a buffered log write.
        access_log /config/log/nginx/access.log;
    
        # Includes virtual hosts configs.
        #include /etc/nginx/http.d/*.conf;
    
        # WARNING: Don't use this directory for virtual hosts anymore.
        # This include will be moved to the root context in Alpine 3.14.
        #include /etc/nginx/conf.d/*.conf;
    
    
        ##
        # Basic Settings
        ##
    
        client_body_buffer_size 128k;
        keepalive_timeout 65;
        large_client_header_buffers 4 16k;
        send_timeout 5m;
        tcp_nodelay on;
        types_hash_max_size 2048;
        variables_hash_max_size 2048;
        # server_names_hash_bucket_size 64;
        # server_name_in_redirect off;
    
        ##
        # Gzip Settings
        ##
    
        gzip on;
        gzip_disable "msie6";
        # gzip_vary on;
        # gzip_proxied any;
        # gzip_comp_level 6;
        # gzip_buffers 16 8k;
        # gzip_http_version 1.1;
        # gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;
    
        ##
        # nginx-naxsi config
        ##
        # Uncomment it if you installed nginx-naxsi
        ##
    
        #include /etc/nginx/naxsi_core.rules;
    
        ##
        # nginx-passenger config
        ##
        # Uncomment it if you installed nginx-passenger
        ##
    
        #passenger_root /usr;
        #passenger_ruby /usr/bin/ruby;
    
        ##
        # Virtual Host Configs
        ##
        include /config/nginx/site-confs/*;
        #Removed lua. Do not remove this comment
    }
    
    #mail {
    #    # See sample authentication script at:
    #    # http://wiki.nginx.org/ImapAuthenticateWithApachePhpScript
    #
    #    # auth_http localhost/auth.php;
    #    # pop3_capabilities "TOP" "USER";
    #    # imap_capabilities "IMAP4rev1" "UIDPLUS";
    #
    #    server {
    #        listen     localhost:110;
    #        protocol   pop3;
    #        proxy      on;
    #    }
    #
    #    server {
    #        listen     localhost:143;
    #        protocol   imap;
    #        proxy      on;
    #    }
    #}
    
    daemon off;
    pid /run/nginx.pid;
    
    ```

13. `Swag`中的`Nginx`反向代理模块配置文件梳理：

    详见`xmind`脑图：

   ```bash
   # /config/ = ./
   ./nginx/nginx.conf -[包含]-> ./nginx/site-confs/default -[包含]-> ./nginx/proxy-confs/*.subdomain.conf
   ```

   > `xmind`链接：[nginx.conf.xmind](D:\文档中心\【卷·八】创力\Xmind)
   >
   > `xmind`大纲：
   >
   > # nginx.conf
   >
   > ## 全局变量
   >
   > ### user
   >
   > ### worker_processes.conf
   >
   > ### events
   >
   > ## **http**
   >
   > ### 全局变量
   >
   > - mime.types
   > - resolver.conf
   > - client_max_body_size
   > - Sendfile
   > - tcp_nopush
   > - access_log
   > - Basic Settings
   >
   >   - client_body_buffer_size
   >   - keepalive_timeout
   >   - large_client_header_buffers
   >   - send_timeout
   >   - tcp_nodelay
   >   - types_hash_max_size
   >   - variables_hash_max_size
   > - Gzip Settings
   > - **upstrem（负载均衡）**
   >
   > ### nginx-naxsi config
   >
   > ### nginx-passenger config
   >
   > ### Virtual Host Configs
   >
   > - 'site-confs/default'
   >
   >   - error_page
   >   - **server{}**
   >
   >     - listen 80;
   >
   >   - **server{}**
   >
   >     - listen 443 ssl;
   >     - server_name _;
   >     - location / {}
   >
   >   - 'proxy-confs/*.subdomain.conf'
   >
   >     - 'seafile.subdomain.conf'
   >
   >       - **server{ }**
   >
   >         - listen 443 ssl;
   >         - **server_name seafile.*;**
   >         - **location / { }**
   >
   >     - 'bookstack.subdomain.conf'
   >
   > ## mail
   >

【注意事项】：

- 当子域名名称、数量改变时，须改变`swag`的`docker-compose.yml`文件，然后重新安装此`compose`，而不是仅仅重新启动`swag`容器；

- 最好保持`DNS`解析商控制台中的子域名对象与`swag`要申请`SSL`证书的域名对象相一直，避免`swag`启动不成功（盲猜有这种可能性）。如确实不再使用某个子域名，可以在控制台中将其停止解析，并在`ddns-go`处取消动态`dns`解析；

- 如果只是修改反向代理的配置文件，重启`swag`容器即可；

- 尝试申请通配符域名`SSL`证书时，出现不成功的情况，通配符证书在申请时似乎也就是匹配了些常见的子级域名，比如`bin`，`var`，`app`等，并对它们申请`SSL`证书。`swag`日志如下：

  ```bash
  ...
  Sub-domains processed are:  -d app.kingchl.cn -d bin.kingchl.cn -d \
  config.kingchl.cn -d defaults.kingchl.cn -d dev.kingchl.cn \
  -d docker-mods.kingchl.cn -d donate.txt.kingchl.cn -d etc.kingchl.cn \
  -d home.kingchl.cn -d init.kingchl.cn -d lib.kingchl.cn -d libexec.kingchl.cn \
  -d media.kingchl.cn -d mnt.kingchl.cn -d opt.kingchl.cn -d proc.kingchl.cn \
  -d root.kingchl.cn -d run.kingchl.cn -d sbin.kingchl.cn -d srv.kingchl.cn \
  -d sys.kingchl.cn -d tmp.kingchl.cn -d usr.kingchl.cn -d var.kingchl.cn
  ...
  ```

  因此改为对特定的子域名申请`SSL`证书；

- `swag`的`Nginx`模块似乎没有打开`HTTP`访问，即使路由器转发到`80`端口，但是刷不出页面；

- `SSL`证书安装在`swag`的`Nginx`服务器端，所以`Nginx`能够以`HTTP`访问。

  访问方式：https://subdomain.kingchl.cn:port@Nginx；

  但在访问该域名下**其他端口**所在的服务器时，即使域名申请了`SSL`证书，因该服务器端没有安装域名`SSL`证书，此时`HTTPS`是无法响应的，提示：

  > # **此站点的连接不安全**
  >
  > **seafile.kingchl.cn** 发送了无效的响应。
  >
  > 
  >
  > 
  >
  > - [尝试运行 Windows 网络诊断](javascript:diagnoseErrors())。
  >
  > ERR_SSL_PROTOCOL_ERROR

