+++
author = "陈 sir"
categories = ["Keycloak "]
tags = ["Keycloak "]
date = "2019-09-21"
description = "Keycloak SSL Configuration"
featured = "main/title.png"
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "Keycloak 配置SSL 实现HTTPS 访问"
type = "post"

+++


# # 生成keystore.jks
``` bash
keytool -genkeypair -alias keycloak.me -keyalg RSA -keystore keycloak.jks -validity 10950
```

> 注意：CN必须是主机名，也可以是ip但是ip容易变，所以主机名或域名。后面访问的是时候通过在hosts 就文件映射域名到IP, 这里将CN 设置为keycloak.me, 注意alias参数不是设置CN的地方

生成好keycloak.jks 之后 复制到 keycloak-7.0.0\standalone\configuration\ 文件夹内

# 修改standalone.xml 或 standalone-ha.xml 或 domain.xml



具体修哪个文件，看你的安装方式，此处使用standalone.xml

在 <security-realm name="ApplicationRealm"> 下添加

``` xml
<security-realm name="ApplicationRealm">
                <server-identities>
                    <ssl>
                        <keystore path="application.keystore" relative-to="jboss.server.config.dir" keystore-password="password" alias="server" key-password="password" generate-self-signed-certificate-host="localhost" />
                    </ssl>
                </server-identities>
                <authentication>
                    <local default-user="$local" allowed-users="*" skip-group-loading="true" />
                    <properties path="application-users.properties" relative-to="jboss.server.config.dir" />
                </authentication>
                <authorization>
                    <properties path="application-roles.properties" relative-to="jboss.server.config.dir" />
                </authorization>
            </security-realm>
<!-- 添加的部分 -->
            <security-realm name="UndertowRealm">
                <server-identities>
                    <ssl>
                        <keystore path="keycloak.jks" relative-to="jboss.server.config.dir" keystore-password="openstack" alias="keycloak.me" />
                    </ssl>
                </server-identities>
            </security-realm>
<!-- 添加的部分 end -->
```

然后 搜索 `<server name="default-server">`

修改为如下内容,

``` xml
 <server name="default-server">
                <http-listener name="default" socket-binding="http" redirect-socket="https" enable-http2="true" />
     <!-- 修改https 的 security realm 为UndertowRealm， 默认为 ApplicationRealm-->
                <https-listener name="https" socket-binding="https" security-realm="UndertowRealm" enable-http2="true" />
                <host name="default-host" alias="localhost">
                    <location name="/" handler="welcome-content" />
                    <http-invoker security-realm="ApplicationRealm" />
                </host>
            </server>
```

重新启动 standalone.bat



# 修改host文件

添加

``` ini
127.0.0.1 keycloak.me
```



访问https://keycloak.me:8443/auth