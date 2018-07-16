### Cetus在fedora上使用mariadb-devel库的编译
#### 1 fedora编译报错

```
[ 47%] Building C object src/CMakeFiles/mysql-chassis-proxy.dir/network-mysqld.c.o
In file included from /src/github.com/Lede-Inc/cetus/src/network-mysqld.c:61:0:
/src/github.com/Lede-Inc/cetus/src/network-mysqld.c: In function ‘proxy_self_create_auth’:
/src/github.com/Lede-Inc/cetus/src/network-mysqld-packet.h:105:32: error: ‘CLIENT_BASIC_FLAGS’ undeclared (first use in this function); did you mean ‘CLIENT_DEFAULT_FLAGS’?
 #define COMPATIBLE_BASIC_FLAGS CLIENT_BASIC_FLAGS
                                ^
/src/github.com/Lede-Inc/cetus/src/network-mysqld-packet.h:108:30: note: in expansion of macro ‘COMPATIBLE_BASIC_FLAGS’
 #define CETUS_DEFAULT_FLAGS (COMPATIBLE_BASIC_FLAGS                     \
                              ^~~~~~~~~~~~~~~~~~~~~~
/src/github.com/Lede-Inc/cetus/src/network-mysqld.c:4436:33: note: in expansion of macro ‘CETUS_DEFAULT_FLAGS’
     auth->client_capabilities = CETUS_DEFAULT_FLAGS;
                                 ^~~~~~~~~~~~~~~~~~~
/src/github.com/Lede-Inc/cetus/src/network-mysqld-packet.h:105:32: note: each undeclared identifier is reported only once for each function it appears in
 #define COMPATIBLE_BASIC_FLAGS CLIENT_BASIC_FLAGS
                                ^
/src/github.com/Lede-Inc/cetus/src/network-mysqld-packet.h:108:30: note: in expansion of macro ‘COMPATIBLE_BASIC_FLAGS’
 #define CETUS_DEFAULT_FLAGS (COMPATIBLE_BASIC_FLAGS                     \
                              ^~~~~~~~~~~~~~~~~~~~~~
/src/github.com/Lede-Inc/cetus/src/network-mysqld.c:4436:33: note: in expansion of macro ‘CETUS_DEFAULT_FLAGS’
     auth->client_capabilities = CETUS_DEFAULT_FLAGS;
                                 ^~~~~~~~~~~~~~~~~~~
make[2]: *** [src/CMakeFiles/mysql-chassis-proxy.dir/build.make:63: src/CMakeFiles/mysql-chassis-proxy.dir/network-mysqld.c.o] Error 1
make[1]: *** [CMakeFiles/Makefile2:993: src/CMakeFiles/mysql-chassis-proxy.dir/all] Error 2
make: *** [Makefile:163: all] Error 2
```

#### 2 报错原因

报错信息上看，应该是编译的时候，没有找到`mysql_com.h`头文件，分别查看centos+mysql-devvel和fedora+mariadb-devel文件的路径：

```
# centos + mysql-devel
/usr/include/mysql/mysql_com.h
# fedora + mariadb-devel
/usr/include/mysql/server/mysql_com.h
```

#### 3 解决办法

首先修改CMakeList.txt，使用 # 注释掉以下这一行

```
SET(MYSQL_LIBRARIES libmysql)
```
其次，cmake的时候，指定mariadb的路径

```
cmake ../ -DCMAKE_BUILD_TYPE=Debug -DCMAKE_INSTALL_PREFIX=/home/gaohaitao/cetus_install -DSIMPLE_PARSER=ON -DMYSQL_INCLUDE_DIRS="/usr/include/mysql/server;/usr/include/mysql/mysql" -DMYSQL_LIBRARIES="mariadb" -DMYSQL_LIBRARY_DIRS="/usr/lib64/"

make && make install
```
#### 4 编译警告

虽然上述编译方法可以正常编译通过，但是编译过程会出现警告：

```
[ 90%] Building C object plugins/admin/CMakeFiles/admin.dir/admin-plugin.c.o
In file included from /data/cetus_dev/src/cetus-users.h:25:0,
                 from /data/cetus_dev/plugins/admin/admin-plugin.c:33:
/data/cetus_dev/plugins/admin/admin-plugin.c: In function ‘server_con_init’:
/data/cetus_dev/src/network-mysqld-packet.h:110:29: warning: large integer implicitly truncated to unsigned type [-Woverflow]
 #define CETUS_DEFAULT_FLAGS (COMPATIBLE_BASIC_FLAGS                     \
                             ^
/data/cetus_dev/plugins/admin/admin-plugin.c:96:31: note: in expansion of macro ‘CETUS_DEFAULT_FLAGS’
     challenge->capabilities = CETUS_DEFAULT_FLAGS & (~CLIENT_TRANSACTIONS);
                               ^~~~~~~~~~~~~~~~~~~
```

警告上看是由于类型转换导致的问题。查看mysql-devel和mariadb-devel的`mysql-com.h`文件，发现定义有所不同：

```
# mysql-devel
#define CLIENT_SSL              2048    /* Switch to SSL after handshake */

# mariadb-devel
#define CLIENT_SSL              2048ULL /* Switch to SSL after handshake */
```

因此，应该是mariadb-devel使用了`ULL`造成的warning。

#### 5 深究是否`CLIENT_BASIC_FLAGS`一致

在不同平台16进制打印出了`CLIENT_BASIC_FLAGS`，发现并不一致：

```
# mysql-devel
81fff7df

# mariadb-devel
81bff7de
```

查看`mysql-com.h`文件，发现mysql-devel和mariadb-devel中的宏定义略有区别，才导致输出不正确。

```
# mysql-devel
#define CLIENT_BASIC_FLAGS (((CLIENT_ALL_FLAGS & ~CLIENT_SSL) \
                                               & ~CLIENT_COMPRESS) \
                                               & ~CLIENT_SSL_VERIFY_SERVER_CERT)

#define CLIENT_ALL_FLAGS  (CLIENT_LONG_PASSWORD \
                           | CLIENT_FOUND_ROWS \
                           | CLIENT_LONG_FLAG \
                           | CLIENT_CONNECT_WITH_DB \
                           | CLIENT_NO_SCHEMA \
                           | CLIENT_COMPRESS \
                           | CLIENT_ODBC \
                           | CLIENT_LOCAL_FILES \
                           | CLIENT_IGNORE_SPACE \
                           | CLIENT_PROTOCOL_41 \
                           | CLIENT_INTERACTIVE \
                           | CLIENT_SSL \
                           | CLIENT_IGNORE_SIGPIPE \
                           | CLIENT_TRANSACTIONS \
                           | CLIENT_RESERVED \
                           | CLIENT_RESERVED2 \
                           | CLIENT_MULTI_STATEMENTS \
                           | CLIENT_MULTI_RESULTS \
                           | CLIENT_PS_MULTI_RESULTS \
                           | CLIENT_SSL_VERIFY_SERVER_CERT \
                           | CLIENT_REMEMBER_OPTIONS \
                           | CLIENT_PLUGIN_AUTH \
                           | CLIENT_CONNECT_ATTRS \
                           | CLIENT_PLUGIN_AUTH_LENENC_CLIENT_DATA \
                           | CLIENT_CAN_HANDLE_EXPIRED_PASSWORDS \
                           | CLIENT_SESSION_TRACK \
                           | CLIENT_DEPRECATE_EOF \
)

# mariadb-devel
#define CLIENT_BASIC_FLAGS (((CLIENT_ALL_FLAGS & ~CLIENT_SSL) \
                                               & ~CLIENT_COMPRESS) \
                                               & ~CLIENT_SSL_VERIFY_SERVER_CERT)
                                               
#define CLIENT_ALL_FLAGS  (\
                           CLIENT_FOUND_ROWS | \
                           CLIENT_LONG_FLAG | \
                           CLIENT_CONNECT_WITH_DB | \
                           CLIENT_NO_SCHEMA | \
                           CLIENT_COMPRESS | \
                           CLIENT_ODBC | \
                           CLIENT_LOCAL_FILES | \
                           CLIENT_IGNORE_SPACE | \
                           CLIENT_PROTOCOL_41 | \
                           CLIENT_INTERACTIVE | \
                           CLIENT_SSL | \
                           CLIENT_IGNORE_SIGPIPE | \
                           CLIENT_TRANSACTIONS | \
                           CLIENT_RESERVED | \
                           CLIENT_SECURE_CONNECTION | \
                           CLIENT_MULTI_STATEMENTS | \
                           CLIENT_MULTI_RESULTS | \
                           CLIENT_PS_MULTI_RESULTS | \
                           CLIENT_SSL_VERIFY_SERVER_CERT | \
                           CLIENT_REMEMBER_OPTIONS | \
                           MARIADB_CLIENT_PROGRESS | \
                           CLIENT_PLUGIN_AUTH | \
                           CLIENT_PLUGIN_AUTH_LENENC_CLIENT_DATA | \
                           CLIENT_SESSION_TRACK |\
                           CLIENT_DEPRECATE_EOF |\
                           CLIENT_CONNECT_ATTRS |\
                           MARIADB_CLIENT_COM_MULTI |\
                           MARIADB_CLIENT_STMT_BULK_OPERATIONS)
                                           
```

#### 6 fedora+mariadb-devel测试功能

在fedora_mariadb-devel环境下编译Cetus，后端使用MySQL-server，简单测试，可以进行正常的SQL通信。由于没有mariadb-server，暂未测试mariadb-server情况。