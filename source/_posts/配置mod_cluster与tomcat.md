title: 配置mod_cluster与tomcat
date: 2014-06-27 23:33:40
categories:
- Java
tags:
- 负载均衡
- mod_cluster
- tomcat
- 分布式
---
{% blockquote %}
mod_cluster 和mod_jk,mod_proxy类似，是一个基于httpd的负载均衡项目。能够代理请求给基于Tomcat的网络服务器集群(支持任何独立的Tomcat，JBoss Web或者JBoss AS)。mod_cluster与 mod_jk和mod_proxy的区别是，mod_cluster为web服务器和httpd服务器之间提供后台通道。web服务器使用后台通道给httpd端提供当前负载信息。 
{% endblockquote %}


# 配置mod_cluster

## 下载mod_cluster
下载集成了mod_cluster模块的apache httpd server [mod_cluster-1.2.6.Final-linux2-x64-ssl.tar.gz](http://downloads.jboss.org/mod_cluster/1.2.6.Final/linux-x86_64/mod_cluster-1.2.6.Final-linux2-x64-ssl.tar.gz)

## 安装步骤
* mkdir -p /tmp/apache
* tar -xzvf mod_cluster-1.2.6.Final-linux2-x64-ssl.tar.gz -C /tmp/apache
* cd /tmp/apache
* cp -r opt/* /opt

<!-- more -->

## 修改配置文件
* 打开配置文件/opt/jboss/httpd/httpd/conf/httpd.conf
* 移动到文件末尾
* 修改标签&lt;IfModule manager_module>&lt;/IfModule>中的内容
```
<IfModule manager_module>
	Listen $ip:$port
	ManagerBalancerName mycluster
	<VirtualHost $ip:$port>
		<Location />
			Order deny,allow
			Deny from all
			Allow from all
		</Location>

		KeepAliveTimeout 300
		MaxKeepAliveRequests 0
		ServerAdvertise on
		EnableMCPMReceive
		AllowDisplay on

		<Location /mod_cluster_manager>
			SetHandler mod_cluster-manager
			Order deny,allow
			Deny from all
			Allow from all
		</Location>
	</VirtualHost>
</IfModule>
```  
* 根据你的机器情况，更改ip和port

## 生成证书 
* 总共需要生成三个证书文件
	* server.crt
	* server.key
	* server-ca.crt
* 将这些证书文件置于/opt/jboss/httpd/httpd/conf/

## 配置SSL 
* 打开/opt/jboss/httpd/httpd/conf/httpd.conf
* 移动至文件末尾
* 增加两个VirtualHost，其中一个用来将http请求redirect到https请求，另一个用来处理SSL连接
```
Listen 80
<VirtualHost *:80>
	RewriteEngine on
	RewriteCond %{SERVER_PORT} 80
	RewriteRule ^{.*} https://%{SERVER_NAME}%{REQUEST_URI} [R,L]
</VirtualHost>

Listen 443
<VirtualHost *:443>
	<Location />
		Order deny,allow
		Deny from all
		Allow from all
	</Location>

	SSLEngine on
	SSLCertificateFile conf/server.crt
	SSLCertificateKeyFile conf/server.key
	SSLCACertificateFile conf/server-ca.crt
</VirtualHost>
```

# 配置Tomcat

## 下载jar包 

下载mod_cluster提供的java bundles [mod_cluster-parent-1.2.6.Final-bin.tar.gz](http://downloads.jboss.org/mod_cluster/1.2.6.Final/linux-x86_64/mod_cluster-parent-1.2.6.Final-bin.tar.gz)

## 安装插件 
* mkdir -p /tmp/bundles
* tar -xzvf mod_cluster-parent-1.2.6.Final-bin.tar.gz -C /tmp/bundles
* cd /tmp/bundles
* cp JBossWeb-Tomcat/lib/*.jar $CATALINA_BASE/lib
* cd $CATALINA_BASE/lib
* 删除无用jar包
	* 如果你的tomcat版本是6    
	rm mod_cluster-container-tomcat7-1.2.6.Final.jar
	* 如果你的tomcat版本是7    
	rm mod_cluster-container-tomcat6-1.2.6.Final.jar

## 修改配置文件 
* 打开$CATALINA_BASE/conf/server.xml
* 在标签&lt;Server>&lt;/Server>中，加入一个Listener
```
<Listener className="org.jboss.modcluster.container.catalina.standalone.ModClusterListener" proxyList="$load_balancer_ip:$load_balancer_port" />
```
* 根据mod_cluster宿主机器信息更改$load_balancer_ip和$load_balancer_port
* 通过在&lt;Engine>&lt;/Engine>标签中添加属性jvmRoute，可以指定该server的名称
```
<Engine name="Catalina" defaultHost="localhost" jvmRoute="myTomcat">
```

# 启动

## 启动mod_cluster 
/opt/jboss/httpd/sbin/apachectl start
## 启动tomcat 
$CATALINA_BASE/bin/startup.sh

## 添加应用服务器 
如果你想为这个负载均衡器添加更多的应用服务器，即tomcat实例，你只需要简单的重复**配置Tomcat **一节的步骤。mod_cluster这边不需要做任何事情，它会自动检测到新添加的应用服务器

## 检查运行状况 
启动之后，可以在浏览器中输入$load_balancer_ip:load_balancer_port/mod_cluster_manager来查看整个系统的运行状况，如下图所示
{% img /img/mod_cluster.jpg 900 %}
