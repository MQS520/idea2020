### 一. 生成SSL自签证书
        自签证书就是自己生成的证书,免费的,不支持部署浏览器的,支持浏览器的就是收费的,需要购买,
    这里因为是本地测试,所以就用的自签证书,买的证书可以跳过证书生成部分.
1. 安装OpenSSL
    
    OpenSSL是生成SSL的工具,这里是在Win10下安装的,下载的windows 64位的,直接下一步安装.
    然后在环境变量的path添加OpenSSL安装的bin路径即可
    [下载链接](http://slproweb.com/products/Win32OpenSSL.html)

2. 本地测试域名
   
   由于是在本机测试所以也没有域名,但是可以通过修改hosts文件模拟域名
   
   `hosts文件在C:\Windows\System32\drivers\etc 目录下,打开添加`
   `127.0.0.1    demo.test.com`
   
3. 开始生成证书
   * 生成RSA私钥 
      
      des3算法,1024位强度,server.key 秘钥文件名
      
      `openssl genrsa -des3 -out server.key 1024`
      
   * 生成CSR(证书签名请求)
   
   `openssl req -new -key server.key -out server.csr`
   
   注意 : Common Name必须和域名保持一致

    ```shell script
    Country Name (2 letter code) [AU]:CN
    State or Province Name (full name) [Some-State]:Beijing
    Locality Name (eg, city) []:Beijing
    Organization Name (eg, company) [Internet Widgits Pty Ltd]:joyios
    Organizational Unit Name (eg, section) []:info technology
    Common Name (e.g. server FQDN or YOUR name) []:demo.test.com   #这一项必须和你的域名一致
    Email Address []:liufan@joyios.com
    ```
    * 删除私钥中的密码
    
    `openssl rsa -in server.key -out server.key`

    * 生成自签名证书
    
    `openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt`
    
###### 这时候证书已经生成好了.包含3个文件:server.key | server.csr | server.crt

### 二. 配置Nginx
* 放置证书
 
    打开nginx的conf目录,创建keys目录,将生成的证书(3个文件)放入keys目录中
    
* 修改nginx.conf

    正常http请求改成https时无需变动,而websocket需要将协议 `ws` 改为 `wss`
   
```shell script
    server {
        listen       80;
        server_name  demo.test.com;
 
	    rewrite ^(.*)$ https://${server_name}$1 permanent;
 
        location / {
            proxy_pass http://www.xxx.com:8080;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Port $server_port;
        }
 
    }
 
    server {
	    listen       443;
        server_name  demo.test.com;
        ssl on;
        #配置证书的路径
        ssl_certificate      keys/server.crt;
        ssl_certificate_key  keys/server.key;
        ssl_session_timeout  5m;
        ssl_ciphers  HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers  on;
        
        # 普通的https请求
        location / {
             #配置转发到8080端口
            proxy_pass http://demo.test.com:8080;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Port $server_port;
        }
        
        # WebSocket 请求
	    location /websocketChat {
            proxy_pass http://demo.test.com:8080;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
        
        # WebSocket 请求
        location /websocketAudio {
            proxy_pass http://demo.test.com:8080;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
 
    }
```

* 重启nginx

    `nginx -s reload`
    

### 三. 测试

    浏览器打开: demo.test.com, 即可识别SSl证书,因为证书没有认证所以浏览器会拦截,点击选择信任,
    即可访问到8080端口项目中
