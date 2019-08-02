Mysql5.7基于日志主从复制

1、先下载 mysql源安装包

        yum -y install wget
        wget https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm   
        yum -y localinstall mysql57-community-release-el7-11.noarch.rpm              ##安装mysql源   
        yum -y install mysql-community-server                                        ##在线安装Mysql
        systemctl restart mysqld
        ##设置开机启动
        systemctl enable mysqld
        systemctl daemon-reload

        ##修改root本地登录密码 mysql安装完成之后，在/var/log/mysqld.log文件中给root生成了一个临时的默认密码。
         vi /var/log/mysqld.log
         mysql -uroot -p
         mysql> set global validate_password_policy=0; ##修改密码的强度验证
         mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY '12345678';
        
2、在master服务器上新建用户，登录mysql后执行以下语句

create user 'dba'@'192.168.25.%' identified by '123456'；   #新建用户,允许登录的ip地址段:192.168.25.% 用户名:dba 密码:123456
grant replication slave on *.* to dba@'192.168.25.%';    #赋权限     

3、配置/etc/my.conf 配置文件

        # For advice on how to change settings please see
        # http://dev.mysql.com/doc/refman/5.7/en/server-configuration-defaults.html
        [mysqld]
        #
        # Remove leading # and set to the amount of RAM for the most important data
        # cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
        # innodb_buffer_pool_size = 128M
        #
        # Remove leading # to turn on a very important data integrity option: logging
        # changes to the binary log between backups.
        # log_bin
        #
        # Remove leading # to set options mainly useful for reporting servers.
        # The server defaults are faster for transactions and fast SELECTs.
        # Adjust sizes as needed, experiment to find the optimal values.
        # join_buffer_size = 128M
        # sort_buffer_size = 2M
        # read_rnd_buffer_size = 2M
        datadir=/var/lib/mysql
        socket=/var/lib/mysql/mysql.sock


        port=4306
        collation-server = utf8mb4_unicode_ci
        character-set-server = utf8mb4
        init-connect ='SET NAMES utf8mb4'
        slow_query_log = ON
        long_query_time = 0.5
        expire_logs_days=20
        sql_mode = STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION

        default-time-zone = +8:00

        #mysql最大连接数
        max_connections = 600

        #某台host连接错误次数等于max_connect_errors（默认10） ，主机'host_name'再次尝试时被屏蔽。可有效反的防止dos攻击
        max_connect_errors = 1000

        #当我们的join是ALL,index,rang或者Index_merge的时候使用的buffer。 实际上这种join被称为FULL JOIN
        join_buffer_size = 128M

        #读入缓冲区的大小，将对表进行顺序扫描的请求将分配一个读入缓冲区，MySQL会为它分配一段内存缓冲区
        read_buffer_size = 16M
        #随机读缓冲区大小，当按任意顺序读取行时（列如按照排序顺序）将分配一个随机读取缓冲区，进行排序查询时，MySQL会首先扫描一遍该缓冲，以避免磁盘搜索，提高查询速度
        read_rnd_buffer_size = 32M

        #ORDER BY 或者GROUP BY 操作的buffer缓存大小
        innodb_sort_buffer_size = 64M


        # Disabling symbolic-links is recommended to prevent assorted security risks
        symbolic-links=0

        log-error=/var/log/mysqld.log
        pid-file=/var/run/mysqld/mysqld.pid

        server-id = 1
        log-bin = mysql-bin                           #打开二进制功能,MASTER主服务器必须打开此项
        max_binlog_size=1024M                          #binlog单文件最大值
        replicate-ignore-db = mysql                    #忽略不同步主从的数据库
        replicate-ignore-db = information_schema
        replicate-ignore-db = performance_schema
        replicate-ignore-db = sys
        replicate-ignore-db = test

##从库操作指令

4、备份master 数据(在从库也可以加参数执行)

    mysqldump --single-transaction --master-data=2 --triggers --routines --all-databases -uroot -p > all.sql 
    
5、Slaver数据库备份数据还原

       mysql -uroot -p < all.sql
       
6、more all.sql 可以看到文件名和起始位置

change master to master_host='192.168.1.1',master_port=4306, master_user='repl',master_password='123456789', master_log_file='mysql-bin.000038',master_log_pos=419500193;

7、启动 slave

start slave;
show slave status \G;
