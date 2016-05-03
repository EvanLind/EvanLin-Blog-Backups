Django 

Django 是一个基于 Python 的开源 Web 应用程序框架，其 MVC 开发的方法，把代码的定义和数据访问的方法（模型）与请求逻辑 （控制器）还有用户接口（视图）分开来，同时将对数据库的操作接口进行封装，支持 sqlite3, MySQL, PostgreSQL等数据库。使得Django 应用程序很简单。开发阶段可使用Django框架自带的一个开发 Web 服务器，使用python manage.py runsercver即可开启服务器。但是这个框架不适合在生产环境中使用，因此需要进一步将 Django 应用程序部署到 Web。在本文中，您将了解到使用nginx和uWSGI在Ubuntu上部署Django。

 

uWSGI & Nginx

WGSI 全称 Web Server Gateway Interface，或者 Python Web Server Gateway Interface ，是为 Python 语言定义的 Web 服务器和 Web 应用程序或框架之间的一种简单而通用的接口。自从 WSGI 被开发出来以后，许多其它语言中也出现了类似接口。当本地有多个 web 服务，有 Python、php、java 编写的，对于同一端口的监听时就必须有一个负责转发的服务了。虽然只但是 uwsgi 对于静态资源处理的并不是很好，一是性能问题，二是各种 HTTP 请求缓存头，则需要与 Nginx进行搭配。
Nginx 是一款轻量级的Web 服务器/反向代理服务器及电子邮件（IMAP/POP3）代理服务器，并在一个BSD-like 协议下发行。由俄罗斯的程序设计师Igor Sysoev所开发，供俄国大型的入口网站及搜索引擎Rambler（俄文：Рамблер）使用。其特点是占有内存少，并发能力强，事实上nginx的并发能力确实在同类型的网页服务器中表现较好。与uWSGI搭配，uWSGI 处理动态数据，Nginx 处理静态文件，索引文件以及自动索引。
 
部署环境

nginx:  nginx -v  -->  nginx version: nginx/1.4.6 (Ubuntu)
uWSGI:  uwsgi --version --> 2.0.12
Ubuntu:  cat /etc/issue --> Ubuntu 14.04.1 LTS \n \l
Python:  python -V --> Python 2.7.6
Django:  python -c "import django;print(django.VERSION)" --> (1, 8, 4, 'final', 0)
 

部署过程（参考uWSGI的文档）

1. 安装nginx

sudo apt-get install nginx

启动、停止和重启

sudo /etc/init.d/nginx start
sudo /etc/init.d/nginx stop
sudo /etc/init.d/nginx restart
或者

sudo service nginx start
sudo service nginx stop
sudo service nginx restart
2. uWSGI安装

apt-get install python-dev
pip install uwsgi
3. 测试Django项目是否正常

python manage.py runserver 0.0.0.0:8000
4. 测试运行uwsgi

uwsgi --http :8001 --chdir /home/web/EvanLin/  --module EvanLin.wsgi
#chdir:工程目录  module:指的是 chdir 下的/EvanLin/wsgi.py文件
5. 部署static文件

在工程目录下的 setting.py 里添加 STATIC_ROOT

STATIC_ROOT = os.path.join(BASE_DIR, "static/")
然后执行：

STATIC_ROOT = os.path.join(BASE_DIR, "static/")
6. 配置nginx

将uwsgi_params文件拷贝到项目文件夹下。uwsgi_params文件在/etc/nginx/目录下
在项目文件夹下创建文件mysite_nginx.conf,填入并修改下面内容：
 # mysite_nginx.conf
 # the upstream component nginx needs to connect to
 upstream django {
 # server unix:///path/to/your/mysite/mysite.sock; # for a file socket
 server 127.0.0.1:8001; # for a web port socket (we'll use this first)
 }
 # configuration of the server
 server {
 # the port your site will be served on
 listen 8000;
 # the domain name it will serve for
 server_name .example.com; # 你的域名或是IP地址
 charset utf-8;
 # max upload size
 client_max_body_size 75M; # adjust to taste
 # Django media
 location /media {
 alias /path/to/your/mysite/media; # Django 的 media 文件夹
 }
 location /static {
 alias /path/to/your/mysite/static; # Django 的 static 文件夹
 }
 # Finally, send all non-media requests to the Django server.
 location / {
 uwsgi_pass django;
 include /path/to/your/mysite/uwsgi_params; # 刚才拷贝的uwsgi_params文件路径
 }
 }
这个文件配置 nginx 从文件系统中使用 media 和 static 文件作为服务，同时相应 django 的 request 在 /etc/nginx/sites-enabled 目录下创建本文件的连接，使nginx能够使用它：

sudo ln -s ~/path/to/your/mysite/mysite_nginx.conf /etc/nginx/sites-enabled/
 
7. 用 UNIX socket 启动 Django 应用

对mysite_nginx.conf做如下修改：

server unix:///path/to/your/mysite/mysite.sock; # for a file socket
# server 127.0.0.1:8001; # for a web port socket (we'll use this first)
重启nginx：

sudo /etc/init.d/nginx start # 若重启失败，通过sudo nginx -t 查错
并运行:

uwsgi --socket mysite.sock --chdir /home/web/mysite/  --module mysite.wsgi --chmod-socket=666 &
8. 多站部署

与部署当个 Django 应用相同，请参照上文。

其中，mysite_nginx.conf 文件修改如下，标红部分需修改，路径为你当前工程目录。其中 listen 监听的端口可以相同，nginx根据 server_name .example.com;来启动不同的 uswgi 文件。

 # mysite_nginx.conf
 # the upstream component nginx needs to connect to
 upstream django {
 server unix:///path/to/your/mysite/mysite.sock; # for a file socket
 # server 127.0.0.1:8001; # for a web port socket (we'll use this first)
 }
 # configuration of the server
 server {
 # the port your site will be served on
 listen 80;
 # the domain name it will serve for
 server_name .example.com; # 你的域名或是IP地址
 charset utf-8;
 # max upload size
 client_max_body_size 75M; # adjust to taste
 # Django media
 location /media {
 alias /path/to/your/mysite/media; # Django 的 media 文件夹
 }
 location /static {
 alias /path/to/your/mysite/static; # Django 的 static 文件夹
 }
 # Finally, send all non-media requests to the Django server.
 location / {
 uwsgi_pass django;
 include /path/to/your/mysite/uwsgi_params; # 刚才拷贝的uwsgi_params文件路径
 }
 }
重启nginx：

sudo /etc/init.d/nginx start # 若重启失败，通过sudo nginx -t 查错
并运行:

uwsgi --socket mysite.sock --chdir /home/web/mysite/  --module mysite.wsgi --chmod-socket=666 & 
如有任何疑问请留言

EvanLin
著作权归作者所有，转载请联系作者获得授权，并标注原文链接。