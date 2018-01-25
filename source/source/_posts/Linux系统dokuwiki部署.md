---
title: Linux系统dokuwiki部署
date: 2017-12-28 15:17:21
tags: dokuwiki部署
---

##### 前提
DokuWiki是一个开源wiki引擎程序，运行于PHP环境下。所以在部署安装DokuWiki之前，需要确认待部署的机器上已有PHP环境。

##### dokuwiki项目部署

1. 在官网下载dokuwiki包，并解压

```
//包下载
wget -c http://download.dokuwiki.org/src/dokuwiki/dokuwiki-stable.tgz

//解压。解压后将解压后的包放在目录/var/www/下，并把包命名dokuwiki。在部署Nginx时，需要用到该目录。如需修改，则需要修改Nginx配置及执行run.sh 的参数。
tar -zvxf dokuwiki-stable.tgz
```

2. 变更权限

<!-- more -->

```
# chown -R apache:apache /var/www/dokuwiki
# find /var/www/dokuwiki -type f -exec chmod 0600 \{\} \;
# find /var/www/dokuwiki -type d -exec chmod 0700 \{\} \;
```
其中的 `apache` 是启动 PHP服务的角色。在我配置的机器中，PHP服务启动的信息为：

```

[resin@node2 ~]$ ps -ef | grep php
root       907     1  0 11月13 ?      00:01:54 php-fpm: master process (/etc/ph-fpm.conf)
apache    2377   907  0 11月13 ?      00:24:54 php-fpm: pool www
apache    2378   907  0 11月13 ?      00:25:04 php-fpm: pool www
apache    2379   907  0 11月13 ?      00:25:00 php-fpm: pool www
apache    2380   907  0 11月13 ?      00:25:06 php-fpm: pool www
apache    2381   907  0 11月13 ?      00:24:52 php-fpm: pool www
apache    2421   907  0 11月13 ?      00:24:57 php-fpm: pool www
apache    2426   907  0 11月13 ?      00:24:59 php-fpm: pool www
apache    2440   907  0 11月13 ?      00:24:55 php-fpm: pool www
apache    2441   907  0 11月13 ?      00:25:00 php-fpm: pool www
apache   17168   907  0 11月14 ?      00:20:38 php-fpm: pool www
apache   20291   907  0 11月13 ?      00:24:13 php-fpm: pool www
apache   24251   907  0 11月15 ?      00:14:23 php-fpm: pool www
root     24805     1  0 12月19 ?      00:03:15 php artisan queue:work --daemon
resin    25475 24887  0 10:00 pts/0    00:00:00 grep --color=auto php
apache   25918   907  0 11月17 ?      00:09:14 php-fpm: pool www
apache   25919   907  0 11月17 ?      00:09:13 php-fpm: pool www
apache   25920   907  0 11月17 ?      00:09:12 php-fpm: pool www
```


4. 配置Nginx

```
server {
    listen       80;
    server_name   dokuwiki.domain.com;
    access_log  /var/logs/nginx/dokuwiki.log  main;

    location ~ /plugins/.+\.(js|css|blade.php)$ {
        root   /var/www/dokuwiki;
    }

    location / {
        root   /var/www/dokuwiki;
        index  index.html index.htm index.php;
           if (!-e $request_filename) {
                rewrite  ^/(.*)$  /index.php/$1  last;
               break;
           }
    }

    location ~.*\.php.* {
       root           /var/www/dokuwiki;
       set $script $uri;
       set $path_info "/";
       if ($uri ~ "^(.+?\.php)(/.+)$"){
           set $script $1;
           set $path_info $2;
        }
        if (-d $request_filename) {
          rewrite ^/(.*)([^/])$ http://$host/$1$2/ permanent;
        }
        fastcgi_pass 127.0.0.1:9000;
        include fastcgi.conf;
        fastcgi_index index.php?IF_REWRITE=1;
       fastcgi_param PATH_INFO $path_info;
       fastcgi_param SCRIPT_NAME $script;
     }

}
```

5. 页面安装
通过访问 `http:dokuwiki.domain.com/install.php`进行相关配置。配置好后，删除install.php文件。

```
# cd /var/www/dokuwiki
# rm -i install.php
```



##### 参考文章
[Dokuwiki中文网](http://www.dokuwiki.com.cn/)

[Dokuwiki官网](https://www.dokuwiki.org/dokuwiki#)

[linux下安装 Dokuwiki](http://www.dokuwiki.com.cn/217.html)


