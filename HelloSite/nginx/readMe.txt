
dotnet new razor -o HelloSite


Publish to a Linux Production Environment
By Sourabh Shirhatti


Publish and copy over the app
Configure the app for a framework-dependent deployment.

Run dotnet publish from the development environment to package an app into a directory (for example, bin/Release/<target_framework_moniker>/publish) that can run on the server:

console

Copy
dotnet publish --configuration Release



In this guide, we will cover setting up a production-ready ASP.NET environment on an Ubuntu 14.04 Server.

We will take an existing ASP.NET Core application and place it behind a reverse-proxy server. We will then setup the reverse-proxy server to forward requests to our Kestrel web server.

Additionally we will ensure our web application runs on startup as a daemon and configure a process management tool to help restart our web application in the event of a crash to guarantee high availability.

Prerequisites
Access to an Ubuntu 14.04 Server with a standard user account with sudo privilege.

An existing ASP.NET Core application.

Copy over your app
Run dotnet publish from your dev environment to package your application into a self-contained directory that can run on your server.

Before we proceed, copy your ASP.NET Core application to your server using whatever tool (SCP, FTP, etc) integrates into your workflow. Try and run the app and navigate to http://<serveraddress>:<port> in your browser to see if the application runs fine on Linux. I recommend you have a working app before proceeding.

[!NOTE] You can use Yeoman to create a new ASP.NET Core application for a new project.

Configure a reverse proxy server
A reverse proxy is a common setup for serving dynamic web applications. The reverse proxy terminates the HTTP request and forwards it to the ASP.NET application.

Why use a reverse-proxy server?
Kestrel is great for serving dynamic content from ASP.NET, however the web serving parts aren’t as feature rich as full-featured servers like IIS, Apache or Nginx. A reverse proxy-server can allow you to offload work like serving static content, caching requests, compressing requests, and SSL termination from the HTTP server. The reverse proxy server may reside on a dedicated machine or may be deployed alongside an HTTP server.

For the purposes of this guide, we are going to use a single instance of Nginx that runs on the same server alongside your HTTP server. However, based on your requirements you may choose a different setup.

Install Nginx
sudo apt-get install nginx
[!NOTE] If you plan to install optional Nginx modules you may be required to build Nginx from source.

We are going to apt-get to install Nginx. The installer also creates a System V init script that runs Nginx as daemon on system startup. Since we just installed Nginx for the first time, we can explicitly start it by running

sudo service nginx start
At this point you should be able to navigate to your browser and see the default landing page for Nginx.

Configure Nginx
We will now configure Nginx as a reverse proxy to forward requests to our ASP.NET application

We will be modifying the /etc/nginx/sites-available/default, so open it up in your favorite text editor and replace the contents with the following.

server {
    listen 80;
    location / {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection keep-alive;
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
This is one of the simplest configuration files for Nginx that forwards incoming public traffic on your port 80 to a port 5000 that your web application will listen on.

Once you have completed making changes to your nginx configuration you can run sudo nginx -t to verify the syntax of your configuration files. If the configuration file test is successful you can ask nginx to pick up the changes by running sudo nginx -s reload.

Monitoring our Web Application
Nginx will forward requests to your Kestrel server, however unlike IIS on Windows, it does not mangage your Kestrel process. In this tutorial, we will use supervisor to start our application on system boot and restart our process in the event of a failure.

Installing supervisor
sudo apt-get install supervisor
[!NOTE] supervisor is a python based tool and you can acquire it through pip or easy_install instead.

Configuring supervisor
Supervisor works by creating child processes based on data in its configuration file. When a child process dies, supervisor is notified via the SIGCHILD signal and supervisor can react accordingly and restart your web application.

To have supervisor monitor our application, we will add a file to the /etc/supervisor/conf.d/ directory.

/etc/supervisor/conf.d/hellomvc.conf

[program:hellomvc]
command=/usr/bin/dotnet /var/aspnetcore/HelloMVC/HelloMVC.dll
directory=/var/aspnetcore/HelloMVC/
autostart=true
autorestart=true
stderr_logfile=/var/log/hellomvc.err.log
stdout_logfile=/var/log/hellomvc.out.log
environment=HOME=/var/www/,ASPNETCORE_ENVIRONMENT=Production
user=www-data
stopsignal=INT
stopasgroup=true
killasgroup=true
Once you are done editing the configuration file, restart the supervisord process to change the set of programs controlled by supervisord.

sudo service supervisor stop
sudo service supervisor start
Start our web application on startup
In our case, since we are using supervisor to manage our application, the application will be automatically started by supervisor. Supervisor uses a System V Init script to run as a daemon on system boot and will susbsequently launch your application. If you chose not to use supervisor or an equivalent tool, you will need to write a systemd or upstart or SysVinit script to start your application on startup.

Viewing logs
Supervisord logs messages about its own health and its subprocess' state changes to the activity log. The path to the activity log is configured via the logfile parameter in the configuration file.

sudo tail -f /var/log/supervisor/supervisord.log
You can redirect application logs (STDOUT and STERR) in the program section of your configuration file.

tail -f /var/log/hellomvc.out.log
Securing our application
Enable AppArmor
Linux Security Modules (LSM) is a framework that is part of the Linux kernel since Linux 2.6 that supports different implementations of security modules. AppArmor is a LSM that implements a Mandatory Access Control system which allows you to confine the program to a limited set of resources. Ensure AppArmor is enabled and properly configured.

Configuring our firewall
Close off all external ports that are not in use. Uncomplicated firewall (ufw) provides a frontend for iptables by providing a command-line interface for configuring the firewall. Verify that ufw is configured to allow traffic on any ports you need.

sudo apt-get install ufw
sudo ufw enable

sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
Securing Nginx
The default distribution of Nginx doesn't enable SSL. To enable all the security features we require, we will build from source.

Download the source and install the build dependencies
# Install the build dependencies
sudo apt-get update
sudo apt-get install build-essential zlib1g-dev libpcre3-dev libssl-dev libxslt1-dev libxml2-dev libgd2-xpm-dev libgeoip-dev libgoogle-perftools-dev libperl-dev

# Download nginx 1.10.0 or latest
wget http://www.nginx.org/download/nginx-1.10.0.tar.gz
tar zxf nginx-1.10.0.tar.gz
Change the Nginx response name
Edit src/http/ngx_http_header_filter_module.c

static char ngx_http_server_string[] = "Server: Your Web Server" CRLF;
static char ngx_http_server_full_string[] = "Server: Your Web Server" CRLF;
Configure the options and build
The PCRE library is required for regular expressions. Regular expressions are used in the location directive for the ngx_http_rewrite_module. The http_ssl_module adds HTTPS protocol support.

Consider using a web application firewall like ModSecurity to harden your application.

./configure
--with-pcre=../pcre-8.38
--with-zlib=../zlib-1.2.8
--with-http_ssl_module
--with-stream
--with-mail=dynamic
Configure SSL
Configure your server to listen to HTTPS traffic on port 443 by specifying a valid certificate issued by a trusted Certificate Authority (CA).

Harden your security by employing some of the practices suggested below like choosing a stronger cipher and redirecting all traffic over HTTP to HTTPS.

Adding an HTTP Strict-Transport-Security (HSTS) header ensures all subsequent requests made by the client are over HTTPS only.

Do not add the Strict-Transport-Security header or chose an appropriate max-age if you plan to disable SSL in the future.

Add /etc/nginx/proxy.conf configuration file.

[!code-nginxMain]

Edit /etc/nginx/nginx.conf configuration file. The example contains both http and server sections in one configuration file.

[!code-nginxMain]










最好的方式是使用进程守护工具
supervisor
sudo apt-get install supervisor

	
I have decided to use supervisor to monitor and manage the process. Here is an excellent article on getting it set up.

It allows simple control over specific dotnet apps like this:
-----------重启站点----------
supervisorctl stop MyWebsiteName
supervisorctl start MyWebsiteName

-----------为进程守护 写配置文件----------
/etc/supervisor/conf.d/bookManage.conf

[program:bookManage]
command=/usr/bin/dotnet /home/chen/webroot/BookManage1.1/bin/Debug/netcoreapp2.0/publish/BookManage.dll
directory=/home/chen/webroot/BookManage1.1/bin/Debug/netcoreapp2.0/publish/
autostart=true
autorestart=true
stderr_logfile=/var/log/BookManage.err.log
stdout_logfile=/var/log/BookManage.out.log
environment=HOME=/var/www/,ASPNETCORE_ENVIRONMENT=Production
user=www-data
stopsignal=INT
stopasgroup=true
killasgroup=true

-----------------------关于进程守护 Supervisor的介绍-----------
重启进程守护  加载新建的配置
sudo service supervisor stop
sudo service supervisor start
sudo service supervisor status

