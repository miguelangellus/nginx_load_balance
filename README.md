# Nginx: Load Balancer
### Requirements
- Nginx
- Docker
- Docker Compose
- VS Code

### Scenario
We'll create three Nginx machine containers:
1. The main server - NLB_Server
2. Sever One - NLB_Node1
3. Server Two - NLB_Node2

We'll have the NLB_Server set up as an HTTPS reverse proxy to intermediate a proxy service for the client requests that redirects to their balance servers NLB_Node1 and NLB_Node2 that in turn deliver the server's response back to the client.

## Hands-on
```
mkdir ~/Desktop/nginx_server_balance && \
code ~/Desktop/nginx_server_balance
```
Now let's go ahead and work into the VS Code integrated terminal to create and set up a docker-compose.yaml file
```
cat <<BOT1 > docker-compose.yml
version: '3'

services:

  nginx-server:
    image: nginx:1.21.6-alpine
    container_name: NLB_Server
    ports:
      - "8081:80"
BOT1
```
After docker-compose is created and set up, run:
```
docker compose up -d
```
In order to see the running images.
```
docker compose ps
```
The output should look like this:
```
miguelangellus@ubuntu-labs:~/Desktop/nginx_server_balance$ sudo docker compose up -d
[+] Running 1/1
 ⠿ Container NLB_Server  Started                                                                                          0.4s
miguelangellus@ubuntu-labs:~/Desktop/nginx_server_balance$ sudo docker compose ps
NAME                COMMAND                  SERVICE             STATUS              PORTS
NLB_Server          "/docker-entrypoint.…"   nginx-server        running             0.0.0.0:8081->80/tcp, :::8081->80/tcp
```
As you can see, the Nginx docker image was properly pulled to your machine and it's running as well. Just open your web browser and type http://localhost:8081, a Nginx welcome page should show for you.
Here is the [source](https://hub.docker.com/_/nginx) of the official Nginx Docker image, in which you'll find all the instructions about the needed parameters to compose your .yaml file.  


1.Now let's install a shell script service in order to have access to the Alpine's terminal, let's take the Bash.
```
sudo docker compose exec nginx-server apk add bash
```
- output:
  ```
  [sudo] password for miguelangellus: 
  fetch https://dl-cdn.alpinelinux.org/alpine/v3.15/main/x86_64/APKINDEX.tar.gz
  fetch https://dl-cdn.alpinelinux.org/alpine/v3.15/community/x86_64/APKINDEX.tar.gz
  (1/2) Installing readline (8.1.1-r0)
  (2/2) Installing bash (5.1.16-r0)
  Executing bash-5.1.16-r0.post-install
  Executing busybox-1.34.1-r5.trigger
  OK: 27 MiB in 44 packages
  ```
2.Now we are able to get into the NLB_Server container bash terminal, under the service named as nginx-server
```
sudo docker compose exec nginx-server bash
```
- output:
  ```
  miguelangellus@ubuntu-labs:~/Desktop/nginx_server_balance$ sudo docker compose exec nginx-server bash
  bash-5.1# |
  ```
3.Once into the terminal, let's investigate and set up the config files located at /etc/nginx path, but before we have to install at least one editor

```
apk update && apk add nano
```
- output:
  ```
  bash-5.1# apk update && apk add nano
  fetch https://dl-cdn.alpinelinux.org/alpine/v3.15/main/x86_64/APKINDEX.tar.gz
  fetch https://dl-cdn.alpinelinux.org/alpine/v3.15/community/x86_64/APKINDEX.tar.gz
  v3.15.4-52-gaf7b2b3e8c [https://dl-cdn.alpinelinux.org/alpine/v3.15/main]
  v3.15.4-53-g5a491f1187 [https://dl-cdn.alpinelinux.org/alpine/v3.15/community]
  OK: 15861 distinct packages available
  OK: 27 MiB in 45 packages
  ```

## Let's take a look at these two nginx config files:
```
cat /etc/nginx/nginx.conf
```
- output:
  ```
  bash-5.1# cat /etc/nginx/nginx.conf 

  user  nginx;
  worker_processes  auto;

  error_log  /var/log/nginx/error.log notice;
  pid        /var/run/nginx.pid;

  events {
      worker_connections  1024;
  }

  http {
      include       /etc/nginx/mime.types;
      default_type  application/octet-stream;

      log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                        '$status $body_bytes_sent "$http_referer" '
                        '"$http_user_agent" "$http_x_forwarded_for"';

      access_log  /var/log/nginx/access.log  main;

      sendfile        on;
      #tcp_nopush     on;

      keepalive_timeout  65;

      #gzip  on;

      include /etc/nginx/conf.d/*.conf;
  }
  ```
Let's see the next.
```
cat /etc/nginx/conf.d/default.conf
```
- output:
  ```
  bash-5.1# cat /etc/nginx/conf.d/default.conf
  server {
      listen       80;
      listen  [::]:80;
      server_name  localhost;

      #access_log  /var/log/nginx/host.access.log  main;

      location / {
          root   /usr/share/nginx/html;
          index  index.html index.htm;
      }

      #error_page  404              /404.html;

      # redirect server error pages to the static page /50x.html
      #
      error_page   500 502 503 504  /50x.html;
      location = /50x.html {
          root   /usr/share/nginx/html;
      }

      # proxy the PHP scripts to Apache listening on 127.0.0.1:80
      #
      #location ~ \.php$ {
      #    proxy_pass   http://127.0.0.1;
      #}

      # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
      #
      #location ~ \.php$ {
      #    root           html;
      #    fastcgi_pass   127.0.0.1:9000;
      #    fastcgi_index  index.php;
      #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
      #    include        fastcgi_params;
      #}

      # deny access to .htaccess files, if Apache's document root
      # concurs with nginx's one
      #
      #location ~ /\.ht {
      #    deny  all;
      #}
  }
  ```
So we're going to get a tweak to this file for a better understanding by editing it and deleting all of the unnecessary lines. By the way, leave enabled the `access_log` located at line six, uncomenting it by just deleting the # (hashtag), we'll need to get access to that logs later.  
```
nano /etc/nginx/conf.d/default.conf
```
- Ater that the default.conf file will look like this:
  ```
  bash-5.1# cat /etc/nginx/conf.d/default.conf
  server {
      listen       80;
      listen  [::]:80;
      server_name  localhost;

      access_log  /var/log/nginx/host.access.log  main;

      location / {
          root   /usr/share/nginx/html;
          index  index.html index.htm;
      }

      #error_page  404              /404.html;

      # redirect server error pages to the static page /50x.html
      #
      error_page   500 502 503 504  /50x.html;
      location = /50x.html {
          root   /usr/share/nginx/html;
      }
  }
  ```
These files is the core of Nginx, after making changes it's a good practice to drop a nginx linter that searchs for syntax errors.
```
bash-5.1# nginx -t
...
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
After success we have to save these last changes.
```
bash-5.1# nginx -s reload
...
2022/04/22 19:53:58 [notice] 66#66: signal process started
```
Now type exit or ctrl+d to close the bash terminal 
```
bash-5.1# exit
exit
miguelangellus@ubuntu-labs:~/Desktop/nginx_server_balance$ 
```

Now let's add to docker-compose a configuration to create two more nginx machines, NLB_Node1 and NLB_Node2. We won't need allow the browser access, so we don't need to bind the port, only the main server should have access to those machines.
```
cat <<COD2 >> docker-compose.yml

  nginx-node1:
    image: nginx:1.21.6-alpine
    container_name: NLB_Node1
    ports:
      - "80"
      
  nginx-node2:
    image: nginx:1.21.6-alpine
    container_name: NLB_Node2
    ports:
      - "80"
COD2
```
Lets climb the newest services.
```
sudo docker compose up nginx-node1 nginx-node2 -d
```
- output
  ```
  miguelangellus@ubuntu-labs:~/Desktop/nginx_server_balance$ sudo docker compose up nginx-node1 nginx-node2 -d
  [sudo] password for miguelangellus: 
  [+] Running 2/2
   ⠿ Container NLB_Node2  Started                                                                                           0.5s
   ⠿ Container NLB_Node1  Started                                                                                           0.6s
  ```
Let's see if it's running for us.
```
sudo docker compose ps
```
- output
  ```
  miguelangellus@ubuntu-labs:~/Desktop/nginx_server_balance$ sudo docker compose ps
  NAME                COMMAND                  SERVICE             STATUS              PORTS
  NLB_Node1           "/docker-entrypoint.…"   nginx-node1         running             0.0.0.0:49157->80/tcp, :::49157->80/tcp
  NLB_Node2           "/docker-entrypoint.…"   nginx-node2         running             0.0.0.0:49156->80/tcp, :::49156->80/tcp
  NLB_Server          "/docker-entrypoint.…"   nginx-server        running             0.0.0.0:8081->80/tcp, :::8081->80/tcp
  miguelangellus@ubuntu-labs:~/Desktop/nginx_server_balance$ 
  ```
Let's make changes to their respective apk repositories after that we're going to add the bash scripting terminal and the nano editor.
```
sudo docker compose exec nginx-node1 apk update && \
sudo docker compose exec nginx-node2 apk update && \
sudo docker compose exec nginx-node1 apk add nano bash && \
sudo docker compose exec nginx-node2 apk add nano bash
```
- output
  ```
  miguelangellus@ubuntu-labs:~/Desktop/nginx_server_balance$ sudo docker compose exec nginx-node1 apk update && \
  > sudo docker compose exec nginx-node2 apk update && \
  > sudo docker compose exec nginx-node1 apk add nano bash && \
  > sudo docker compose exec nginx-node2 apk add nano bash
  fetch https://dl-cdn.alpinelinux.org/alpine/v3.15/main/x86_64/APKINDEX.tar.gz
  fetch https://dl-cdn.alpinelinux.org/alpine/v3.15/community/x86_64/APKINDEX.tar.gz
  v3.15.4-52-gaf7b2b3e8c [https://dl-cdn.alpinelinux.org/alpine/v3.15/main]
  v3.15.4-53-g5a491f1187 [https://dl-cdn.alpinelinux.org/alpine/v3.15/community]
  OK: 15861 distinct packages available
  fetch https://dl-cdn.alpinelinux.org/alpine/v3.15/main/x86_64/APKINDEX.tar.gz
  fetch https://dl-cdn.alpinelinux.org/alpine/v3.15/community/x86_64/APKINDEX.tar.gz
  v3.15.4-52-gaf7b2b3e8c [https://dl-cdn.alpinelinux.org/alpine/v3.15/main]
  v3.15.4-53-g5a491f1187 [https://dl-cdn.alpinelinux.org/alpine/v3.15/community]
  OK: 15861 distinct packages available
  (1/3) Installing readline (8.1.1-r0)
  (2/3) Installing bash (5.1.16-r0)
  Executing bash-5.1.16-r0.post-install
  (3/3) Installing nano (5.9-r0)
  Executing busybox-1.34.1-r5.trigger
  OK: 27 MiB in 45 packages
  (1/3) Installing readline (8.1.1-r0)
  (2/3) Installing bash (5.1.16-r0)
  Executing bash-5.1.16-r0.post-install
  (3/3) Installing nano (5.9-r0)
  Executing busybox-1.34.1-r5.trigger
  OK: 27 MiB in 45 packages
  ```
Let's make changes to their respectives index.html files, which can be located in the path `/usr/share/nginx/html/`. 
let's get into the nginx-node1 bash terminal:
```
sudo docker compose exec nginx-node1 bash
```
Now, we can make changes to that.
```
nano /usr/share/nginx/html/index.html
```
- Leave your index.html file like this
  ```
  bash-5.1# cat /usr/share/nginx/html/index.html 
  <!DOCTYPE html>
  <html>
    <head>
    <title>::node1::</title>
    <style>
      html { color-scheme: light dark; }
      body { width: 35em; margin: 0 auto;
      font-family: Tahoma, Verdana, Arial, sans-serif; }
    </style>
    </head>
    <body>
      <h1>Hello from Nginx-node1 Server!!!</h1>
    </body>
  </html>
  ```
Now we can get out of this bash environment.

Let's go to the main server.
```
sudo docker compose exec nginx-server bash
```
Now let's get into the file default.conf going to location and adding a proxy_pass target parameter to point to http://nginx-node1. Also in location we'll find root and index parameters, either you delete them or simply coment with hashtags #. 
```
nano /etc/nginx/conf.d/default.conf
```
- after made the changes, default.conf file should seem like this.
  ```
  bash-5.1# cat /etc/nginx/conf.d/default.conf
  server {
      listen       80;
      listen  [::]:80;
      server_name  localhost;

      access_log  /var/log/nginx/host.access.log  main;

      location / {
         # root   /usr/share/nginx/html;
         # index  index.html index.htm;
         proxy_pass http://nginx-node1;
      }

      #error_page  404              /404.html;

      # redirect server error pages to the static page /50x.html
      #
      error_page   500 502 503 504  /50x.html;
      location = /50x.html {
          root   /usr/share/nginx/html;
      }
  }
  ```
Verify for syntax errors. If it's all right, reload the nginx service and get out of its bash environment.
```
nginx -t
```
```
nginx -s reload
```
```
exit
```
- output
  ```
  bash-5.1# nginx -t
  nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
  nginx: configuration file /etc/nginx/nginx.conf test is successful
  bash-5.1# nginx -s reload
  2022/04/22 05:03:13 [notice] 113#113: signal process started
  bash-5.1# exit
  ```
Then go to your web browser opened to http://localhost:8081.
Now what should show for you is "Hello from Nginx-node1 Server!!!"

Now We're going to make changes to nginx-node2 service. Let's get into the nginx-node2 bash terminal:
```
sudo docker compose exec nginx-node2 bash
```
Now, we can make changes to that.
```
nano /usr/share/nginx/html/index.html
```
- Leave its index.html file like this
  ```
  bash-5.1# cat /usr/share/nginx/html/index.html 
  <!DOCTYPE html>
  <html>
    <head>
    <title>::node2::</title>
    <style>
      html { color-scheme: light dark; }
      body { width: 35em; margin: 0 auto;
      font-family: Tahoma, Verdana, Arial, sans-serif; }
    </style>
    </head>
    <body>
      <h1>Hello from Nginx-node2 Server!!!</h1>
    </body>
  </html>
  ```
After that, get out of its bash environment.

Now we're going to make the needed changes to the main nginx default.conf file going to location and setting up the proxy_pass parameter to point to http://nginx-node2 as well. In order to do that we're going to add both nginx-node1 and nginx-node2 as upstreams.
```
sudo docker compose exec nginx-server bash
```
```
nano /etc/nginx/conf.d/default.conf
```
- New configuration to `default.conf` file.
  ```
  bash-5.1# cat /etc/nginx/conf.d/default.conf 
  upstream mynodes {
      server nginx-node1;
      server nginx-node2;
  }

  server {
      listen       80;
      listen  [::]:80;
      server_name  localhost;

      access_log  /var/log/nginx/host.access.log  main;

      location / {
         # root   /usr/share/nginx/html;
         # index  index.html index.htm;
         proxy_pass http://mynodes;
      }

      #error_page  404              /404.html;

      # redirect server error pages to the static page /50x.html
      #
      error_page   500 502 503 504  /50x.html;
      location = /50x.html {
          root   /usr/share/nginx/html;
      }
  }
  ```
Verify for syntax errors. If it's all right, reload the nginx service and get out of its bash environment.
```
nginx -t
```
```
nginx -s reload
```
```
exit
```
Again, back to the web browser session opened to http://localhost:8081.
Now what should show for you are both "Hello from Nginx-node1 Server!!!" and "Hello from Nginx-node2 Server!!!" toggling between themselves after reloading the web browser.
## Access Logs and pings
```
tail -f /var/log/nginx/host.access.log
```
```
bash-5.1# tail -f /var/log/nginx/host.access.log 
172.20.0.1 - - [23/Apr/2022:00:15:35 +0000] "GET / HTTP/1.1" 200 267 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/100.0.4896.127 Safari/537.36" "-"
172.20.0.1 - - [23/Apr/2022:00:15:36 +0000] "GET / HTTP/1.1" 200 267 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/100.0.4896.127 Safari/537.36" "-"
172.20.0.1 - - [23/Apr/2022:00:15:37 +0000] "GET / HTTP/1.1" 200 267 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/100.0.4896.127 Safari/537.36" "-"
172.20.0.1 - - [23/Apr/2022:00:15:37 +0000] "GET / HTTP/1.1" 200 267 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/100.0.4896.127 Safari/537.36" "-"
172.20.0.1 - - [23/Apr/2022:00:34:18 +0000] "GET / HTTP/1.1" 200 267 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/100.0.4896.127 Safari/537.36" "-"
172.20.0.1 - - [23/Apr/2022:00:34:32 +0000] "GET / HTTP/1.1" 200 267 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/100.0.4896.127 Safari/537.36" "-"
172.20.0.1 - - [23/Apr/2022:00:34:42 +0000] "GET / HTTP/1.1" 200 267 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/100.0.4896.127 Safari/537.36" "-"
172.20.0.1 - - [23/Apr/2022:00:34:47 +0000] "GET / HTTP/1.1" 200 267 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/100.0.4896.127 Safari/537.36" "-"
172.20.0.1 - - [23/Apr/2022:00:34:48 +0000] "GET / HTTP/1.1" 200 267 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/100.0.4896.127 Safari/537.36" "-"
172.20.0.1 - - [23/Apr/2022:00:34:50 +0000] "GET / HTTP/1.1" 200 267 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/100.0.4896.127 Safari/537.36" "-"
^C
bash-5.1# ping nginx-server
PING nginx-server (172.20.0.2): 56 data bytes
64 bytes from 172.20.0.2: seq=0 ttl=64 time=0.051 ms
64 bytes from 172.20.0.2: seq=1 ttl=64 time=0.087 ms
64 bytes from 172.20.0.2: seq=2 ttl=64 time=0.088 ms
64 bytes from 172.20.0.2: seq=3 ttl=64 time=0.087 ms
^C
--- nginx-server ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.051/0.078/0.088 ms
bash-5.1# ping nginx-node1
PING nginx-node1 (172.20.0.4): 56 data bytes
64 bytes from 172.20.0.4: seq=0 ttl=64 time=0.133 ms
64 bytes from 172.20.0.4: seq=1 ttl=64 time=0.118 ms
64 bytes from 172.20.0.4: seq=2 ttl=64 time=0.110 ms
64 bytes from 172.20.0.4: seq=3 ttl=64 time=0.123 ms
64 bytes from 172.20.0.4: seq=4 ttl=64 time=0.136 ms
^C
--- nginx-node1 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max = 0.110/0.124/0.136 ms
bash-5.1# ping nginx-node2
PING nginx-node2 (172.20.0.3): 56 data bytes
64 bytes from 172.20.0.3: seq=0 ttl=64 time=0.123 ms
64 bytes from 172.20.0.3: seq=1 ttl=64 time=0.125 ms
64 bytes from 172.20.0.3: seq=2 ttl=64 time=0.107 ms
64 bytes from 172.20.0.3: seq=3 ttl=64 time=0.130 ms
64 bytes from 172.20.0.3: seq=4 ttl=64 time=0.116 ms
^C
--- nginx-node2 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max = 0.107/0.120/0.130 ms
bash-5.1# ping localhost
PING localhost (127.0.0.1): 56 data bytes
64 bytes from 127.0.0.1: seq=0 ttl=64 time=0.081 ms
64 bytes from 127.0.0.1: seq=1 ttl=64 time=0.085 ms
64 bytes from 127.0.0.1: seq=2 ttl=64 time=0.098 ms
64 bytes from 127.0.0.1: seq=3 ttl=64 time=0.086 ms
^C
--- localhost ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.081/0.087/0.098 ms
bash-5.1# 
```
