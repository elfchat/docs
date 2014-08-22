# WebSocket Server
If you have UNLIM version of ElfChat, you can use WebSocket version. To do this you need to go to ElfChat Admin configuration and select *Server Type*: `WebSocket Server`. 
Now you need to start daemon. Go to your server using the SSH and run following command:
```
$ php ~/chat/app/server.php 
```

Now deamon is running.

## Deployment
This tutorial will give you recommendations on what you should do to setup your production environment to run a WebSocket server. None of these are required, but recommended. Please note this page serves as an introduction and boilerplate setup for each technoloogy; each topic has wealths of more detailed resources available.

## ulimit
Before you run command you should probably update your ulimit. What is this? It's a unix security feature that limits the number of open file descriptors one running process can have open at a time. We want to increase this as each client connecting to your server opens a file descriptor. You should update this number to make sure the number of clients you expect never exceeds your ulimit. Remember, when you update your ulimit, that change only affects the current shell session. Before you run you application make sure to execute this command:
```
$ ulimit -n 10000
```

## Libevent
Libevent is an asynchronous event driven C library that is meant to replace default event loop API on your system. It presents a common API that will utilize any event loop that your kernel uses. To set your system to use Libevent you will need to install the system library, the development tools, and the PHP extension:
```
$ sudo apt-get install libevent libevent-dev
$ sudo pecl install libevent
```
The event loop that PHP uses by default through stream_select uses the old, slow poll mechanism. By using Libevent PHP will use the faster epoll or kqueue, drastically improving concurrency (handling many connections quickly). Once Libevent is installed you don't need to change anything in your script. ElfChat will automatically use Libevent if it's available.

## XDebug
Disable the XDebug extension. Make sure it is commented out of your php.ini file. XDebug is fantastic for development, but destroys performance in production.

## Server Configuration
There's a couple things you need to know when deploying a WebSocket server that impact how you set up your server architecture:

* Apache can not handle or even pass-through WebSocket traffic (as of this writing)
* Some aggressive proxies will block traffic that isn't on port 80 or 443 (not many, research your target audience)

Given these issues with WebSockets we have three choices on how to architect:

* Run your website and WebSocket server on the same machine using port 8080 for WebSockets and take the chance client proxies won't block traffic
* Run your WebSocket server on its own server on port 80 under a subdomain (sock.example.com)
* Put a reverse proxy (Nginx) in front of your webserver and WebSocket server

The first two options are fairly easy with the second being a decent option if you can afford a second server. 

### Nginx Proxy Configuration

```
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen   80;

    ...

    location /server/ {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
    }

    ...
}
        
```

And start ElfChat demon on 8080 port.


## Supervisor
When running ElfChat in production it's highly recommended launching it from suporvisord. Suporvisor is a daemon that launches other processes and ensures they stay running. If for any reason your long running ElfChat application halted the supervisor daemon would ensure it starts back up immediately. Supervisor can be installed with any of the following tools: pip, easy_install, apt-get, yum. 

Create `/etc/supervisor/conf.d/elfchat.conf ` file:

```
[program:elfchat]
command                 = bash -c "ulimit -n 10000 && /usr/bin/php /home/chat/app/server.php --host=localhost --port=8080 --path=/server/"
process_name            = ElfChat
numprocs                = 1
autostart               = true
autorestart             = true
user                    = elfet
stdout_logfile          = /home/chat/app/open/out.log
stdout_logfile_maxbytes = 1MB
stderr_logfile          = /home/chat/app/open/err.log
stderr_logfile_maxbytes = 1MB
```

If you're only going to user supervisor to run your WebSocket application you can now run it with the command:

```
$ sudo supervisorctl restart
```

## Links
* [LibEvent](http://libevent.org/)
* [Libevent PHP extension](http://pecl.php.net/package/libevent)
* [Supervisor Configuration](http://supervisord.org/configuration.html)
