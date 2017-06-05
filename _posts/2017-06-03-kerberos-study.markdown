---
layout: post
title:  "kerberos心得"
author: interma
date:   2017-06-03
published: true
---


## kerberos心得

总结一下最近项目中学习到的kerberos知识，尚未实验确认的标明*guessed*

### 基础
先看这篇文章：http://www.tuicool.com/articles/6j2uQn ，简单来说：CPN(client principal)拿着一个可以长期访问SPN（server principal）的ticket，server通过这个ticket来完成对client的身份认证。

### tips
1. kerberos只负责认证，一定要**明确CPN和SPN**
  * 每个PN都还有自己的密钥（keytab文件），只有自己和KDC知道
  * server和KDC是不交互的，client给的ticket已经用server的密钥加密了，
2. spnego，ticket在http header中传递
  * *guessed*：SPN是基于约定：HTTP/FQDN@REALM 
  * 留意FQDN，要写对服务地址，否则可能出现checksum error
3. 一个帐号下的kinit是否可以init多个CPN？
  * 不能，后一个会destroy之前的CPN
  * 如果有此类需求，可以考虑使用区分cache文件并设置KRB5CCNAME环境变量，举例
  ```
[root@test1 gpadmin]# klist
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: rangerlookup@HAWQ.PIVOTAL.COM
Valid starting     Expires            Service principal
05/31/17 01:18:07  06/01/17 01:18:07  krbtgt/HAWQ.PIVOTAL.COM@HAWQ.PIVOTAL.COM
     renew until 05/31/17 01:18:07
[root@test1 gpadmin]# klist -c /tmp/k_s
Ticket cache: FILE:/tmp/k_s
Default principal: spnego@HAWQ.PIVOTAL.COM
Valid starting     Expires            Service principal
05/31/17 01:08:14  06/01/17 01:08:14  krbtgt/HAWQ.PIVOTAL.COM@HAWQ.PIVOTAL.COM
     renew until 05/31/17 01:08:14
  ```
4. 编码
  * java中的JDBC
    * 连接字符串中写明了CPN和SPN，可以参考[HDB Doc](http://hdb.docs.pivotal.io/220/hawq/clientaccess/kerberos.html#topic9)
  * java中的UserGroupInformation类
    * 访问支持kerberos的服务，用它特别方便，代码可以参考[RPS kerberos support](https://github.com/interma/interma-hawq/commit/09439bad6dbcfa5819b4f4bc695ee50982b81b68)
    * 尽量使用loginUserFromKeytab，而不是使用kinit（好处是某些场合下不用改代码），这样同一个帐号下的多个进程不互相影响
    * 不过loginUserFromKeytab后的ticket是无法通过klist看到的，*guessed*：可能保存在内存中？
  * C中用krb5-devel，核心是下边2个函数
    * [krb5_recvauth](https://web.mit.edu/Kerberos/krb5-devel/doc/appdev/refs/api/krb5_recvauth.html)
    * [krb5_sendauth](https://web.mit.edu/Kerberos/krb5-devel/doc/appdev/refs/api/krb5_sendauth.html)
5. 更多tips，见reference  

### In HAWQ
* HAWQ Core  
  * 都在src/interfaces/libpq/下，grep krb5_recvauth和krb5_sendauth，分别是服务端和客户端的代码
  * SPN假定好了为postgres，可以grep代码with_krb_srvnam
  * 支持kerberos用户
    * 打开pg_hda.conf支持（gss method）
    * create role以授权
  * CPN**只能**使用kinit指定，krb5_sendauth有个参数指定它
  * libhdfs和libyarn TODO
* RPS
  * RPS->ranger admin: CPN在rps.properties中配置，通过spnego访问ranger secure webservice
  * ranger lookup->HAWQ: 使用JDBC

### reference
* http://www.tuicool.com/articles/6j2uQn
* http://jxy.me/2015/04/16/kerberos-tips/
* http://tech.meituan.com/hadoop-security-practice.html
* https://www.ibm.com/developerworks/websphere/library/techarticles/0809_lansche/0809_lansche.html
* http://henning.kropponline.de/2016/02/14/a-secure-hdfs-client-example/
* http://jpmens.net/2012/06/23/postgresql-and-kerberos/
