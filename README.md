# 集群部署说明 - 项目套件使用说明
```
常规姿势：/bin/sh /usr/local/iam/bin/standalone.sh 默认指向的配置文件为 standalone\configuration\standalone.xml ， 该姿势只能满足单机跑服务。
默认情况下，多个Keycloak实例的用户、角色等数据信息以及Session是不同步的，所以部署集群重点是要解决以下2个问题：
集群内Keycloak实例的Realm、客户端、用户、角色等数据共享或者同步（Mysql）
集群内Keycloak实例的Session共享或者同步（Keycloak使用[Infinispan]缓存）
```

```
本文实现将集群封装到docker内部，通过docker + mysql + tcp + jdbc_ping发现方式启动集群
```

## 集群步骤
```
名词解释：
 standalone-cluster.xml ：是指我们的集群配置,，通过standalone-ha.xml进行修改而来
 standalone.sh ：keycloak启动脚本
```

### 1.生成docker镜像
这里我们集群的启动方式为：./bin/standalone.sh --server-config=standalone-cluster.xml
这里我们通过一个基础镜像，然后把代码打入镜像中, 生成镜像（如果你不想打成镜像，也没关系，本质是上面的启动方式）
```
FROM docker.bs58i.baishancloud.com/base/ubi8-minimal
WORKDIR /app
COPY .  /app
CMD ./bin/standalone.sh --server-config=standalone-cluster.xml
```

### 2.配置docker-compose
```
接下来配置docker-compose 这边几个关键的暴露端口
image: 这边是上面生成的镜像文件，具体看你自己文件
./configuration/standalone-cluster.xml:/app/standalone/configuration/standalone-cluster.xml  需要把配置文件映射到docker中
端口说明：
8800 ： jboss.http.port http服务端口
8843： jboss.https.port
7600：jgroups-tcp 关键暴露端口 jgroups发现探测端口
57600：jgroups-tcp-fd 关键暴露端口 jgroups发现探测端口
9900：jboss.management.http.port
```
```
version: '3.7'
services:
  iam-server-wzj1:
    container_name: iam-server-wzj1
    image: registry.xxxxx.com/intl/cluster-edgenext-iam-test:1.0.0
    volumes:
      - /etc/hosts:/etc/hosts
      - ./configuration/standalone-cluster.xml:/app/standalone/configuration/standalone-cluster.xml:rw
      - /logs/iam/log:/app/standalone/log/:rw
    ports:
      - "9900:9900"
      - "8800:8800"
      - "8843:8843"
      - "7600:7600"
      - "57600:57600"
    restart: always
```

### 3.配置文件说明
keycloak 代码目录下/standalone/configuration/  皆为它的配置文件，常规情况下其他我们不需要改，只需要更改standalone-cluster.xml集群配置即可，里面包含了数据库，缓存，集群发现机制
### 4.集群配置修改（关键）
#### 4.1.修改数据库，这边使用mysql，这边只要添加中间那段mysql的就可以，其他相关的不需要动
```
<subsystem xmlns="urn:jboss:domain:datasources:5.0">
    <datasources>
        <datasource jndi-name="java:jboss/datasources/ExampleDS" pool-name="ExampleDS" enabled="true" use-java-context="true" statistics-enabled="${wildfly.datasources.statistics-enabled:${wildfly.statistics-enabled:false}}">
            ....
        </datasource>
        <datasource jndi-name="java:jboss/datasources/KeycloakDS" pool-name="KeycloakDS" enabled="true" use-java-context="true" statistics-enabled="${wildfly.datasources.statistics-enabled:${wildfly.statistics-enabled:false}}">
            <connection-url>jdbc:mysql://172.18.156.111:3306/iam?characterEncoding=UTF-8</connection-url>
            <driver>mysql</driver>
            <security>
                <user-name>数据库账号</user-name>
                <password>数据库密码</password>
            </security>
        </datasource>
        <drivers>
            ....
        </drivers>
    </datasources>
</subsystem>
```

#### 4.2.修改缓存infinispan的集群数量
这边需要将以下数量改成2，我们这边部署的是两个节点
```
<subsystem xmlns="urn:jboss:domain:infinispan:9.0">
    <cache-container name="keycloak">
        .....
        <distributed-cache name="sessions" owners="2"/>
        <distributed-cache name="authenticationSessions" owners="2"/>
        <distributed-cache name="offlineSessions" owners="2"/>
        <distributed-cache name="clientSessions" owners="2"/>
        <distributed-cache name="offlineClientSessions" owners="2"/>
        <distributed-cache name="loginFailures" owners="2"/>
        <local-cache name="authorization">
            <object-memory size="10000"/>
        </local-cache>
        .....

```
#### 4.3.使用jgroups tcp jdbc_ping自动发现
```
这边只要将以下模块，替换成我下面那段就可以。说明
这边stack=tcp表示用的tcp
修改external_addr为宿主机的IP
协议protocol采用的是JDBC_PING，初始化的initialize_sql用的是MySQL的，如果是其他数据库，用其他数据库的写法
max_join_attempts表示尝试次数，如果自动发现不了，几次后就不尝试
```
```
<subsystem xmlns="urn:jboss:domain:jgroups:7.0">
    <channels default="ee">
        <channel name="ee" stack="tcp" cluster="ejb"/>
    </channels>
    <stacks>
        <stack name="tcp">
            <transport type="TCP" socket-binding="jgroups-tcp">
                <property name="external_addr">本机IP</property>
            </transport>
            <!-- <socket-protocol type="MPING" socket-binding="jgroups-mping"/> -->
            <protocol type="JDBC_PING">
                <property name="datasource_jndi_name">java:jboss/datasources/KeycloakDS</property>
                <property name="initialize_sql">CREATE TABLE IF NOT EXISTS JGROUPSPING (own_addr varchar(200) NOT NULL, cluster_name varchar(200) NOT NULL, updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP, ping_data varbinary(5000) DEFAULT NULL, PRIMARY KEY (own_addr, cluster_name))
                </property>
            </protocol>
            <protocol type="MERGE3"/>
            <protocol type="FD_SOCK" socket-binding="jgroups-tcp-fd"/>
            <protocol type="FD_ALL"/>
            <protocol type="VERIFY_SUSPECT"/>
            <protocol type="pbcast.NAKACK2"/>
            <protocol type="UNICAST3"/>
            <protocol type="pbcast.STABLE"/>
            <protocol type="pbcast.GMS">
                <property name="max_join_attempts">5</property>
            </protocol>
            <protocol type="MFC"/>
            <protocol type="FRAG2"/>
        </stack>
    </stacks>
</subsystem>

```
#### 4.4.配置proxy-address-forwarding=true
```
不管使用的是什么模式的负载均衡，我们都有可能在业务中需要使用到客户访问的IP地址。
我们在特定的业务中需要获取到用户的ip地址来进行一些操作，比如记录用户的操作日志，如果不能够获取到真实的ip地址的话，则可能使用错误的ip地址。还有就是根据ip地址进行的认证或者防刷工作。
如果我们在服务之前使用了反向代理服务器的话，就会有问题。所以需要我们配置反向代理服务器，保证X-Forwarded-For和X-Forwarded-Proto这两个HTTP header的值是有效的。
然后服务器端就可以从X-Forwarded-For获取到客户的真实ip地址了。
添加如下：
```
```
<subsystem xmlns="urn:jboss:domain:undertow:10.0" default-server="default-server" default-virtual-host="default-host" default-servlet-container="default" default-security-domain="other" statistics-enabled="${wildfly.undertow.statistics-enabled:${wildfly.statistics-enabled:false}}">
    <buffer-cache name="default"/>
    <server name="default-server">
        <ajp-listener name="ajp" socket-binding="ajp"/>
        <http-listener name="default" socket-binding="http" redirect-socket="https" enable-http2="true" proxy-address-forwarding="true"/>
        <https-listener name="https" socket-binding="https" security-realm="ApplicationRealm" enable-http2="true"/>
        <host name="default-host" alias="localhost">
            ....
        </host>
    </server>
```


#### 4.5.修改绑定IP jboss.bind.address
这边因为我是docker，所以统一用0.0.0.0暴露所有哈。 如果非docker的直接写本机IP就可以了，或者127.0.0.1
```
<interfaces>
    <interface name="management">
        <inet-address value="${jboss.bind.address.management:0.0.0.0}"/>
    </interface>
    <interface name="private">
        <inet-address value="${jboss.bind.address.private:0.0.0.0}"/>
    </interface>
    <interface name="public">
        <inet-address value="${jboss.bind.address:0.0.0.0}"/>
    </interface>
</interfaces>
```

### 5.nginx代理
配置nginx代理来做负载均衡
```
server {
    listen       80;
    server_name  account-api.test.com;
    charset utf-8;

    access_log /var/log/nginx/account-api.test.com.log main;
    error_log /var/log/nginx/account-api.test.com.err;

    location / {
        proxy_set_header Host $http_host;
        proxy_pass  http://account-api_server; 
    }
}
upstream account-api_server {
    server 173.18.15.1:8800 weight=1;
    server 173.18.15.2:8800 weight=1;
}

```

### 6.测试结果
```
两边启动服务 ：docker-compose up -d
nginx这边启动就不额外说明
观察日志：
1.数据表JGROUPSPING会生成记录
2.启动会监听端口
3.测试前端登陆，后台登陆，只要都可以正常，那恭喜你，配置成功了
```


