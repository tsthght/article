#### MySQL复制建立过程分析
#### 1 流程分析
主从复制建立的流程大致如下所示：

![8.1.1.png](./images/8.1.1.png)


过程大致分为四个阶段：TCP三次握手建立阶段、MySQL权限认证阶段、获取master信息和dump数据。前两个阶段与普通登录是一样的，重点介绍下后两个阶段的作用。



#### 参考
- [MySQL-Binlog协议详解](https://blog.csdn.net/hj7jay/article/details/56665057)
- [深入解析MySQL replication协议](https://www.jianshu.com/p/5e6b33d8945f)