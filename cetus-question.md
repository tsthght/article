1 SSL的用法
cmake ../ -DCMAKE_BUILD_TYPE=Debug -DCMAKE_INSTALL_PREFIX=/home/gaohaitao/cetus_install -DSIMPLE_PARSER=ON -DWITH_OPENSSL=ON

2 cetus分片中对Join的支持
cetus的join要求：
1 分片表在同一个vdb中
2 join on 后面需要两个表的分片键，有两点考虑：
- 减少数据量
- 保证数据的一致性，必须要join-on的条件是分库键

3 多进程版本cetus，版本升级方案：
kill -USR2 `cat /home/wangbin/github/cetus_install/cetus.pid` ，启动新的cetus，并修改原先的pid文件为old pid文件
kill -WINCH `cat /home/wangbin/github/cetus_install/cetus.pid.oldbin` 旧的cetus不再接受新的连接请求，并且设置成旧的cetus为set maintain true态
kill -QUIT `cat /home/wangbin/github/cetus_install/cetus.pid.oldbin`  旧的cetus退出

需要考虑配置不一致的情况~~
