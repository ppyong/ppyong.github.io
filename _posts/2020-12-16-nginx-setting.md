---
layout: post
title: Nginx 설치와 설정 
---

프로젝트 진행 중 Nginx를 통해 웹서버를 구축하게 되었고 진행 중 제대로 파악을 못해 고생했던 점들이 있어서 추후 확인을 위해 정리하였습니다. 

현재 저희 프로젝트의 대략적인 구조는 아래와 같이 되어있습니다. 

public존에 웹서버를 두고 private존에 앱서버(tomcat)과 DBMS를 두고 있고 웹서버와 앱서버 앞에는 L4를 통해 로드밸런싱하고 있습니다. 

외부 사용자로의 모든 요청은 웹서버를 거치고 이런 요청들 중 정적파일(html, css, js등..)에 대한 요청일 경우 웹서버가 직접 응답을 주고 동적인 요청에 대해서는 

Nginx에서 REVERSE PROXY를 통해 앱서버 앞단의 L4로 넘기고 L4에서 각 앱서버를 통해 처리 후 응답을 주는 형태입니다. 

위 내용중 웹서버인 Nginx에 대한 내용입니다. 

먼저 Nginx 설치부터 진행하도록 하겠습니다. 

+ Nginx 설치를 위한 yum repository 추가 
    
```bash
sudo vi /etc/yum.repos.d/nginx.repo
```
위와 같이 파일을 생성 후 아래 내용을 추가 합니다. 

```conf
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/7/$basearch/
gpgcheck=0
enabled=1
```

+ yum을 통해 Nginx 설치를 진행 합니다. 
    
```bash
sudo yum install -y nginx
```   

+ Nginx 설치가 완료되면 기본적으로 /etc/nginx 경로에 설치가 됩니다. 해당 위치로 이동하여 설정을 진행 합니다. 
위 경로에 nginx.conf 파일의 내용은 아래와 같습니다. 대략 설정 내용의 의미는 주석을 통해 추가하였습니다. 
    
```conf
user  nginx; 
worker_processes  1;
 
error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;
  
events {
    worker_connections  1024;
}
 
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
 
    # 로그 포맷, 로그 설정 시 지정하면 해당 포맷으로 로그를 출력함 
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
 
    # 접근 로그 생성 위치 지정. 위에서 생성한 로그 포맷 (main) 으로 로그를 출력함 
    access_log  /var/log/nginx/access.log  main;
 
    sendfile        on;
    #tcp_nopush     on;
 
    # HTTP의 경우 기본적으로 상태를 가지지 않으므로 매 요청/응답 후 연결을 종료하는데, 바로 종료하지 않고 해당 시간동안 연결을 유지하도록 함
    keepalive_timeout  65;
 
    #gzip  on;
 
    # nginx.conf 에서 설정을 분리하기 위함. Apache의 VirtualHost와 같이 처리하기 위함 
    include /etc/nginx/sites-enabled/*;
}
```

위와 같이 nginx.conf를 설정했다면 기본적으로 /etc/nginx/sites-enabled 파일의 설정을 읽어서 Nginx가 기동되게 됩니다. 

해당 설명에서는 /etc/nginx/sites-available 디렉토리에 설정 파일을 위치하고 각 설정은 VirtualHost로 동작할 수 있게 각 도메인 별로 설정 파일을 관리하며 이중 사용할 속성에 대해서만 /etc/nginx/sites-enabled 로의 소프트링크를 생성하여 동작하게 처리합니다. 

웹서버와 앱서버로의 요청에 대한 도메인 각각 생성할 것이며 웹서버에서 server_name을 통해 어느 서버로의 요청인지 구분하게 됩니다. 

예시로 웹서버 도메인은 "web.ppyong.co.kr", 앱서버 도메인은 "app.ppyong.co.kr"로 진행하도록 하겠습니다. 

+ 앱서버 도메인에 대한 설정을 진행합니다. 

```conf
server {
    listen       80;
    server_name  app.ppyong.co.kr;
 
    access_log  /var/log/nginx/app.ppyong.co.kr.access.log  main; # nginx.conf에 설정한 main 포맷으로 로그를 작성
    error_log   /var/log/nginx/app.ppyong.co.kr.error.log;
 
    location / {
        if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Allow-Origin' "$http_origin";
            add_header 'Access-Control-Allow-Credentials' 'true';
            add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS';
            add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range';
            add_header 'Access-Control-Max-Age' 1728000;
            add_header 'Content-Type' 'text/plain; charset=utf-8';
            add_header 'Content-Length' 0;
            return 204;
        }
 
        add_header 'Access-Control-Allow-Origin' "$http_origin" always;
        add_header 'Access-Control-Allow-Credentials' 'true' always;
        add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS';
        add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range';
        add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Range';
 
        proxy_pass       http://123.123.123.123:8080;
        proxy_set_header Host            $host;
        proxy_set_header X-Forwarded-For $remote_addr;
    }
}
```

위 설정에 대한 대략적인 내용은 web.ppyong.co.kr:80 으로의 요청이 왔을 때 해당 설정을 적용하겠다는 의미이고 access_log 에 설정한 경로로 접근 로그, error_log에 설정한 경로로 에러 로그를 출력 합니다. 

HTTP OPTION Method로 요청이 들어왔을 때는 CORS Preflight 처리를 위해 정해진 응답을 내려주고 그 외의 요청일 경우에는 Proxy로 설정된 123.123.123.123:8080 으로 요청을 넘깁니다. 

서로 다른 도메인에서 Custom 헤더를 설정한 요청일 경우 CORS 이슈가 발생하게 되는데, 해당 이슈 방지를 위해 위 설정과 같이 Origin Header를 추가하고 혹시나 Proxy로 설정한 앱서버에서 2XX Status 값이 아닌 다른 값으로 넘어올 경우에도 해당 헤더를 유지하기 위해 always를 추가 합니다. 

+ 웹서버 도메인에 대한 설정을 진행합니다. 

```conf
server {
    listen       80;
    server_name  web.ppyong.co.kr;
 
    access_log  /var/log/nginx/web.ppyong.co.kr.access.log  main; //nginx.conf에 설정한 main 포맷으로 로그를 작성
    error_log   /var/log/nginx/web.ppyong.co.kr.error.log;
 
    root /usr/share/nginx/html;  # root는 요청 경로 포함하여 지정된 디렉토리에서 찾고, alias는 경로는 제외하고 지정된 디렉토리에서 찾음
 
    location / {
        index  index.html index.htm;
        try_files $uri $uri/ /index.html;  # try_files PATH1 {PATH2} fallback 형식으로 매칭 되지 않는 경로에 대해 PATH1, PATH2 순으로 찾고 실패 시 fallfack 경로의 파일을 보여주게 됩니다.
    }
 
    #location ~* ^.+\.(css|js)$ {
    #    access_log off;
    #    add_header Cache-Control "public";
    #    expires 3600;
    #}
 
    #location ~* ^.+\.(jpg|jpeg|gif|png|swf|ico|zip|tgz|gz|rar|bz2|doc|xls|exe|pdf|ppt|txt|tar|mid|midi|wav|bmp|rtf|mov)$ {
    #    access_log off;
    #    add_header Cache-Control "public";
    #    expires 1d;
    #}
 
    error_page  400 401 402 403 404 405 406 407 408 409 410 411 412 413 414 415 416 417 422 423 424 426 /error.html;
    error_page  500 501 502 503 504 505 506 507 508 510  /error.html;
 
    location = /error.html {
        root    /usr/share/nginx/html/errors;
        internal; # 내부 요청에서만 요효함
    }
}
```

앱서버 설정과 다른 부분만을 설명하자면 
root는 정적인 파일이 존재하는 경로를 뜻합니다. 특정 location 이 아닌 server 설정 아래에 root를 추가하였기 때문에 모든 location에 docroot는 /usr/share/nginx/html 이 됩니다. 

location / 의 경우 web.ppyong.co.kr/ 하위 모든 요청에 대해서 해당 설정으로 동작하겠다는 의미이고 기본 index 파일에 대한 설정과 try_files 을 통해 요청에 해당하는 파일을 찾지 못했을 때 보여줄 화면을 설정하게 됩니다. 

location = /error.html 의 경우에 위와는 다르게 = 연산자로 설정이 되어있는데 경로가 아닌 특정 파일에 대한 요청에 대한 설정 입니다. /error.html 로의 요청에 대한 설정은 /usr/share/nginx/html/errors/error.html 에 해당하는 화면을 보여주게 됩니다. 

또한 internal 설정을 통해 외부에서 유효하지 않고 내부 설정에서만 유효하게 합니다. 

추가로 location 내부에서는 root와 alias로 설정할 수 있는데 

예로 아래와 같이 root로 설정되어 있을 경우 

```conf
location /module {
    root    /usr/share/nginx;
}
```
/usr/share/nginx/module/hello.html 를 보여주게 됩니다. 

예로 아래와 같이 alias로 설정되어 있을 경우 

```conf
location /module {
    alias    /usr/share/nginx;
}
```
/usr/share/nginx/hello.html 를 보여주게 됩니다. 

위에 보시는거와 같이 root의 경우 location에 설정된 경로와 root 경로를 조합하여 응답을 내려주고, alias의 경우에는 요청에서 location에서 설정한 경로를 제외한 경로를 alias에서 찾아 응답을 내려주게 됩니다. 

+ 생성한 설정들에 대해서 sites-enabled 디렉토리에 링크를 생성합니다. 

```bash
ln -s /etc/nginx/sites-available/web.ppyong.co.kr /etc/nginx/sites-enabled/web.ppyong.co.kr
ln -s /etc/nginx/sites-available/app.ppyong.co.kr /etc/nginx/sites-enabled/app.ppyong.co.kr
```

+ 설정을 마쳤으면 service nginx reload 를 통해 설정을 적용합니다. 

