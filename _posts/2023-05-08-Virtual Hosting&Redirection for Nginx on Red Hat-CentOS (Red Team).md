---
layout: post
title: "Virtual Hosting & Redirection for Nginx on Red Hat-CentOS (Red Team)"
author: "Selim Can Mutlu"
categories: sample-posts
tags: [documentation,sample]
permalink: /sample-posts
image: basic-redirector.png
---
Nginx can be used for various purposes, such as a web server, reverse proxy, or load balancer. In this post, I will go through the steps of installing an Nginx web server and configuring simple redirection for TLDs (Top Level Domains) (.nl, .com, .net), along with running the server on a non-default port. You might be asking now why I do not prefer the Apache web server. It's because Nginx outperforms Apache when it comes to serving as a reverse proxy, but the situation is reversed when it comes to functioning as a web server. This installation will be combined with hosting phishing emulation for Red Team engagements in the days to come.
## Some prerequisites 
-   In my environment, I have one CentOS server and one Fedora client.
-   The CentOS web server should be able to resolve both scanmutlu.nl and scanmutlu.net. The same goes for the Fedora client as well.
-   All of these operations require root-level permissions.

### Installing Nginx 
```bash
yum install -y nginx
```
![Alt text](assets/img/1-yuminstall.png)

Installation can be verified with **rpm** command. 
See all relevant files for nginx and see their file types.
```bash
rpm -ql nginx | xargs file
```
![Alt text](assets/img/2-rpmxargs.png)

### Configuring Firewall
Default webserver ports, that is 80, 443, have been opened.
```bash
firewall-cmd --add-port=80/tcp --permanent
firewall-cmd --add-port=443/tcp --permanent
```

![Alt text](assets/img/3-firewall-cmd-add-port.png)

Let the changes be in effect by reloading the firewall.
```bash
firewall-cmd --reload
```
![Alt text](assets/img/4-firewall-cmd--reload.png)
Check out if everything worked out as intended.
```bash
firewall-cmd --list-ports
```

![Alt text](assets/img/5-firewall-cmd--list-ports.png)

### Enabling and Starting Nginx
With enable option service gets started at boot automatically.
```bash
systemctl enable --now nginx
systemctl status nginx
```

![Alt text](assets/img/6-systemctlenable--nownginx.png)
The availability of the service can also be verified by firing up nmap command.

```bash
nmap -sC -sV -p- 192.168.0.128
```
**-sC : run scripts 
-sV : enumerate versions
-p- : scan all ports**

### Changing default root directory to non-default place
I chose to place web files under the /mnt directory, rather than the default document root /usr/share/nginx/html/index.html. I created directories and looked into SELinux permissions respectively, and then realized that the **httpd_sys_content_t** label was not present. This label is essential for the webserver to have context mapping between the nginx process and CentOS, since the process will try to access the file hosted on CentOS. Now, the question might arise as to how we can possibly know which context label to set? Let's check out the default nginx context labels for reference.
```bash
ls -laZ /usr/share/nginx/html/
```
![Alt text](assets/img/9-lsalzusrsharenginxhtml.png)
See **httpd_sys_content_t**

```bash
mkdir -p /mnt/scanmutlu.nl/
mkdir -p /mnt/scanmutlu.net/
ls -lZ /mnt/
```
![Alt text](assets/img/8-lslZmnt.png)

```bash
semanage fcontext -a -t httpd_sys_content_t "/mnt/scanmutlu.net(/.*)?"
restorecon -Rv /mnt/scanmutlu.net
ls -ldZ /mnt/scanmutlu.net/
```
![Alt text](assets/img/10-semanagefcontextrestorecon.png)

```bash
semanage fcontext -a -t httpd_sys_content_t "/mnt/scanmutlu.nl(/.*)?"
restorecon -Rv /mnt/scanmutlu.nl
ls -ldZ /mnt/scanmutlu.nl
```

![Alt text](assets/img/11-semanagenl11.png)
The contents for root directories should be filled out now.
Create index.html under /mnt/scanmutlu.net
```html
<html>
	<head><title>SCANMUTLU.NET</title></head>
<body>
	<p>SCANMUTLU.NET</p>
</body>
</html>
```
Create index.html under /mnt/scanmutlu.nl
```html
<html>
	<head><title>SCANMUTLU.NL</title></head>
<body>
	<p>SCANMUTLU.NL</p>
</body>
</html>
```

### Changing Default port(80) to Non-Default Port(2500)
Both SELinux and firewall configuration should be adjusted accordingly. We need a variance of semanage command that is **semanage port** to set correct context label for the custom port.
See all port context labels related http port;
```bash 
semanage port -l | grep -i http
```
![Alt text](assets/img/12semanageport-lgrep-ihttp.png)
It is evident that the custom port tcp/2500 we are going to connect does not associate with **http_port_t** port context label.  tcp/2500 must be inserted to this group of ports. 
```bash
semanage port -a -t http_port_t -p tcp 2500
semanage port -l | grep -i http
```
![Alt text](assets/img/13semageport-lgrep-ihttp.png)
Now it is time to adjust firewall configuration with port 2500.
```bash
firewall-cmd --add-port=tcp/2500 --permanent
firewall-cmd --reload
firewall-cmd --list-ports
```
![Alt text](assets/img/14-firewall-cmd--add-port=2500tcp.png)

Consequently, **httpd** service should be allowed to be redirected at other websites, as it is by default restricted. **setsebool** command should be used subsequently. 
**-P** argument stands for persistence.
See all booleans with **getsebool -a**
**-a** argument stands for listing all booleans.
```bash
getsebool -a 
setsebool -P httpd_can_network_connect 1
```
![Alt text](assets/img/15setsebool-Phttpd_can_network_connect1.png)

### Adjusting Nginx Configuration File
If you've made it this far, there is one crucial step left, which is the modification of the default configuration file located in **/etc/nginx/nginx.conf**. There are some lines worth mentioning. There are 3 server blocks in total. Since I want to serve over the custom port 2500, I have returned a **403** code in the second server block for port 80. The **proxy_pass** directive is used to convey the semantics of redirection to the **.nl** domain in the third server block.
```
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log notice;
pid /run/nginx.pid;
include /usr/share/nginx/modules/*.conf;
events {
    worker_connections 1024;
}
http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /var/log/nginx/access.log  main;
    sendfile            on;
    tcp_nopush          on;
    keepalive_timeout   65;
    types_hash_max_size 4096;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;
    server {
    	server_name	scanmutlu.nl;
		listen		2500;
    	root		/mnt/scanmutlu.nl/;
    	access_log	/var/log/nginx/scanmutlu.nl/access.log;	
    	error_log	/var/log/nginx/scanmutlu.nl/error.log;
    }
    server {
    	server_name	scanmutlu.nl;
		listen		80;
		return	403;
    }
    server {
    	server_name	scanmutlu.net;
		listen		80;
		location / {
			proxy_pass	http://scanmutlu.nl:2500;
	}
    	access_log	/var/log/nginx/scanmutlu.net/access.log;
		error_log	/var/log/nginx/scanmutlu.net/error.log;
    }
}
```

Check out the syntax of the configuration nginx file.
```bash
nginx -t
```
![Alt text](assets/img/16-nginx-t.png)
Restart the service
```bash
systemctl restart nginx
```

### Confirmation of Nginx Server
I have spinned up Fedora client for testing the webserver.
```url
http://scanmutlu.nl:2500
```
![Alt text](assets/img/17-scanmutlu.nl.png)
```url
http://scanmutlu.net
```
This has successfully redirected to scanmutlu.nl as expected.
![Alt text](assets/img/18-scanmutlu.net.png)

#### References
- https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/deploying_different_types_of_servers/setting-up-and-configuring-nginx_deploying-different-types-of-servers
- https://www.redhat.com/sysadmin/setting-reverse-proxies-nginx