# docker-mysql-initsql

在docker 创建 mysql 容器时，往往需要在创建容器的过程中创建database 实例，代码如下：
```
docker run -d -p 3308:3306 -e MYSQL_ROOT_PASSWORD=123456 -e MYSQL_DATABASE=my_db mysql:latest
```

MYSQL_ROOT_PASSWORD=123456，指定 root 用户名密码 123456
 MYSQL_DATABASE=my_db 创建数据库实例 my_db
***
但有时我们还希望在创建实例的过程中初始化我们的sql脚本，mysql的官方镜像可以支持在容器启动的时候自动执行指定的sql脚本或者shell脚本，我们一起来看看[mysql官方镜像的Dockerfile](https://github.com/docker-library/mysql/blob/7a850980c4b0d5fb5553986d280ebfb43230a6bb/8.0/Dockerfile)，如下图：

![mysql 官方镜像.png](http://upload-images.jianshu.io/upload_images/7548454-a93ecc31b3c162c3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

已经设定了ENTRYPOINT，里面会调用/entrypoint.sh这个脚本，脚本其中一段内容如下图：


![屏幕快照 2017-08-24 下午1.17.39.png](http://upload-images.jianshu.io/upload_images/7548454-bec84dfe7478d69a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


遍历docker-entrypoint-initdb.d目录下所有的.sh和.sql后缀的文件并执行。

原理清楚了，那我们就开始动手操作，思路是将数据库初始化脚本拷贝到docker-entrypoint-initdb.d 目录下，编写Dockerfile 文件，内容如下：

#### fileName: Dockerfile
```
#基础镜像使用 mysql:latest
FROM mysql:latest

#作者
MAINTAINER gjaxx <gjaxx@sohu.com>

#定义会被容器自动执行的目录
ENV AUTO_RUN_DIR /docker-entrypoint-initdb.d

#定义初始化sql文件
ENV INSTALL_DB_SQL init_database.sql

#把要执行的sql文件放到/docker-entrypoint-initdb.d/目录下，容器会自动执行这个sql
COPY ./$INSTALL_DB_SQL $AUTO_RUN_DIR/

#给执行文件增加可执行权限
RUN chmod a+x $AUTO_RUN_DIR/$INSTALL_DB_SQL
```
#### fileName: init_database.sql
```
-- 建库
CREATE DATABASE IF NOT EXISTS my_db default charset utf8 COLLATE utf8_general_ci;

-- 切换数据库
use my_db;

-- 建表
DROP TABLE IF EXISTS `table1`;

CREATE TABLE `table1` (
`id` int(11) unsigned NOT NULL AUTO_INCREMENT,
`name` varchar(20) DEFAULT NULL COMMENT '姓名',
`age` int(11) DEFAULT NULL COMMENT '年龄',
PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

-- 插入数据
INSERT INTO `table1` (`id`, `name`, `age`)
VALUES
(1,'姓名1',10),
(2,'姓名2',11);
```
生成镜像
```
docker build -t init_mysql:0.0.1 .
```
“init_mysql:0.0.1”为镜像名称，“.” 表示Dockerfile在当前路径下，可以通过
```
docker images
```
命令查看本地镜像，结果如下：

![屏幕快照 2017-08-24 下午2.03.27.png](http://upload-images.jianshu.io/upload_images/7548454-2d1528670ec28446.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

根据镜像生成容器
```
docker run --name mysql -p 12345:3306 -e MYSQL_ROOT_PASSWORD=123456 -d init_mysql:0.0.1
```
“--name mysql” 指定容器名字，“-p 12345:3306”指定容器暴漏的端口号，“init_mysql:0.0.1” 镜像名称，通过
```
docker ps
```
命令查看运行中的容器，如下：

![屏幕快照 2017-08-24 下午2.00.33.png](http://upload-images.jianshu.io/upload_images/7548454-f280deda10831557.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以通过
```
docker exec -it CONTAINERID bin/bash
```
命令，进入运行中容器查看初始化脚本是否成功，其中“CONTAINERID” 为容器ID，进入容器后登录mysql，可以看到my_db 已存在，table1已创建，表中已有初始化数据 
```
mysql -uroot -p123456;
show databases;
use my_db;
select * from table1;
```
***

但以上方式有个问题，就是如果有多个sql文件，无法保证执行顺序，这就需要引入 sh 文件，思路是在docker-entrypoint-initdb.d 目录下放置 sh 文件，在 sh 文件中依次执行 sql 文件，编写Dockerfile、install_db.sh、init_database.sql、init_table.sql、init_data.sql 文件，内容如下：
####fileName: Dockerfile
```
#基础镜像使用 mysql:latest
FROM mysql:latest

#作者
MAINTAINER gjaxx <gjaxx@sohu.com>

#定义工作目录
ENV WORK_PATH /usr/local/work

#定义会被容器自动执行的目录
ENV AUTO_RUN_DIR /docker-entrypoint-initdb.d

#定义sql文件名
ENV FILE_0 init_database.sql 
ENV FILE_1 init_table.sql
ENV FILE_2 init_data.sql

#定义shell文件名
ENV INSTALL_DB_SHELL install_db.sh

#创建文件夹
RUN mkdir -p $WORK_PATH

#把数据库初始化数据的文件复制到工作目录下
COPY ./$FILE_0 $WORK_PATH/
COPY ./$FILE_1 $WORK_PATH/
COPY ./$FILE_2 $WORK_PATH/

#把要执行的shell文件放到/docker-entrypoint-initdb.d/目录下，容器会自动执行这个shell
COPY ./$INSTALL_DB_SHELL $AUTO_RUN_DIR/

#给执行文件增加可执行权限
RUN chmod a+x $AUTO_RUN_DIR/$INSTALL_DB_SHELL
```

####fileName：install_db.sh
```
mysql -uroot -p$MYSQL_ROOT_PASSWORD << EOF
source $WORK_PATH/$FILE_0;
source $WORK_PATH/$FILE_1;
source $WORK_PATH/$FILE_2; 
```
####fileName：init_database.sql
```
CREATE DATABASE IF NOT EXISTS my_db default charset utf8 COLLATE utf8_general_ci;
```
####fileName：init_table.sql
```
use my_db;

DROP TABLE IF EXISTS `table1`;

CREATE TABLE `table1` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(20) DEFAULT NULL COMMENT '姓名',
  `age` int(11) DEFAULT NULL COMMENT '年龄',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
####fileName：init_data.sql
```
use my_db;

INSERT INTO `table1` (`id`, `name`, `age`)
VALUES
	(1,'姓名1',10),
	(2,'姓名2',11);
```
生成镜像和容器代码如上，这里就不重复了。
