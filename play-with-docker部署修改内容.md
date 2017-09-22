##### 修改内容
1. 修改了==docker-compose.yml==中 pwd1/2容器的环境变量```GOOGLE_RECAPTCHA_DISABLED="1"```.之前默认为"true"

2. 修改了==haproxy/haproxy.cfg==,将
```
frontend http-in
acl host_localhost hdr(host) localhost
acl host_pwd1 hdr_reg(host) -i ^.*\.?host1\.localhost?:?.*$
acl host_pwd2 hdr_reg(host) -i ^.*\.?host2\.localhost?:?.*$
use_backend all if host_localhost
use_backend pwd1 if host_pwd1
use_backend pwd2 if host_pwd2
```
这部分对访问策略的配置删掉,使用 ```default_backend all```  对任何类型访问都转发到后端pwd1/2:3000上

3. 修改www/下的==bypass.html,index.html,welcome.html==文件中包含被墙的链接地址,在http://www.bootcdn.cn上找到替代链接地址  
```
google被墙地址
- http://ajax.googleapis.com/ajax/libs/angular_material/1.1.0/angular-material.min.css
- https://ajax.googleapis.com/ajax/libs/angularjs/1.5.5/angular.min.js
- https://ajax.googleapis.com/ajax/libs/angularjs/1.5.5/angular-animate.min.js
- https://ajax.googleapis.com/ajax/libs/angularjs/1.5.5/angular-aria.min.js
- https://ajax.googleapis.com/ajax/libs/angular_material/1.1.0/angular-material.min.js
- https://ajax.googleapis.com/ajax/libs/angularjs/1.5.5/angular-messages.min.js
--https://cdnjs.cloudflare.com/ajax/libs/socket.io/1.7.3/socket.io.js
```

```
替代地址
- https://cdn.bootcss.com/angular-material/1.1.0/angular-material.min.css
- https://cdn.bootcss.com/angular.js/1.5.5/angular.min.js
- https://cdn.bootcss.com/angular.js/1.5.5/angular-animate.min.js
- https://cdn.bootcss.com/angular.js/1.5.5/angular-aria.min.js
- https://cdn.bootcss.com/angular-material/1.1.0/angular-material.min.js
- https://cdn.bootcss.com/angular-messages/1.5.5/angular-messages.min.js
--https://cdn.bootcss.com/socket.io/1.7.3/socket.io.js
```
4. 修改 ==pwd/session.go== 文件,在代码185行添加
 ```
 if s == nil {
 		return nil
 	}
```
5. 在阿里云虚拟机和办公室服务器虚拟机上做了一个==tinc vpn==,将阿里云虚拟机和办公室服务器虚拟机互通.

6. 在阿里云虚拟机上搭建一个==haproxy==,前端是阿里云公网IP的80端口,后端是办公室服务器的80端口. 阿里云虚拟机80端口使用```hotstone.com.cn```域名绑定.
```
➜  ~ cat /etc/haproxy/haproxy.cfg
global
	log /dev/log	local0
	log /dev/log	local1 notice
	chroot /var/lib/haproxy
	stats socket /run/haproxy/admin.sock mode 660 level admin
	stats timeout 30s
	user haproxy
	group haproxy
	daemon

	# Default SSL material locations
	ca-base /etc/ssl/certs
	crt-base /etc/ssl/private

	# Default ciphers to use on SSL-enabled listening sockets.
	# For more information, see ciphers(1SSL). This list is from:
	#  https://hynek.me/articles/hardening-your-web-servers-ssl-ciphers/
	ssl-default-bind-ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:ECDH+3DES:DH+3DES:RSA+AESGCM:RSA+AES:RSA+3DES:!aNULL:!MD5:!DSS
	ssl-default-bind-options no-sslv3

defaults
	log	global
	mode	http
	option	httplog
	option	dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
	errorfile 400 /etc/haproxy/errors/400.http
	errorfile 403 /etc/haproxy/errors/403.http
	errorfile 408 /etc/haproxy/errors/408.http
	errorfile 500 /etc/haproxy/errors/500.http
	errorfile 502 /etc/haproxy/errors/502.http
	errorfile 503 /etc/haproxy/errors/503.http
	errorfile 504 /etc/haproxy/errors/504.http
frontend http-in
        bind *:80
        default_backend all
backend all
        mode http  
        balance roundrobin
        server web1 192.168.0.2:80 check
listen admin_stats  
        bind 0.0.0.0:1080               #设置Frontend和Backend的组合体，监控组的名称，按需要自定义名称  
        mode http                       #http的7层模式  
        option httplog                  #采用http日志格式  
        #log 127.0.0.1 local0 err               #错误日志记录  
        maxconn 10                                              #默认的最大连接数  
        stats refresh 30s               #统计页面自动刷新时间  
        stats uri /stats                #统计页面url  
        stats realm XingCloud\ Haproxy  #统计页面密码框上提示文本  
        stats auth admin:admin     #设置监控页面的用户和密码:admin,可以设置多个用户名  
        stats auth  Frank:Frank   #设置监控页面的用户和密码：Frank  
        stats hide-version              #隐藏统计页面上HAProxy的版本信息  
        stats  admin if TRUE       #设置手工启动/禁用，后端服务器(haproxy-1.4.9以后版本)  

```