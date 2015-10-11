#nginx集群

###1.安装nginx

>因为session问题,nginx对session不友好,所以采用阿里的tengine(只是对nginx进行封装,语法和nginx一样)  

######链接:http://tengine.taobao.org/
>版本采用最新版2.1.1

######安装步骤
1. 上传tengine至usr/local目录  
2. 编译tengine  
命令:./configure --prefix=/usr/local/tengine --with-http_stub_status_module --with-http_ssl_module  
说明:将编译后的tengine放到/usr/local/tengine目录下,并且安装ssl模块  
注意点:  
>1. 必须进入tengine的源码目录(/tengine的一级目录)进行编译,否则无法找到tengine  
>2. 编译的目录和源码目录不能相同否则编译会失败!!!!  
>3. linux可能环境没有安装好,这个需要根据具体情况百度  
>4. 如果编译失败的话请删除tengine重试
3. 进入/tengine/sbin/目录下启动tengine  
        nginx #启动  
        nginx -s reload #重启  
        nginx -s stop  #停止
4. 配置集群,进入/usr/local/tengine/conf目录修改nginx.conf配置文件,下面是一个配置好的范例,不需要看
        worker_processes  2;  
        error_log  logs/error.log;
        pid        logs/nginx.pid;
        events {
            worker_connections  1024;
        }
        http {  
            upstream snkefu-web{
                server 218.244.149.110:8080;
                server 218.244.149.110:9081;
	            session_sticky;
        }
	    include       mime.types;
        default_type  application/octet-stream;
        log_format main  '$remote_addr - $remote_user [$time_local] "$request" '
                         '$status $body_bytes_sent "$http_referer" '
                         '"$http_user_agent" "$http_x_forwarded_for"';
        access_log  logs/access.log  main;
        sendfile       on;
        tcp_nopush     on;
        server {
            server_name snkefu.niuhome.com;
            listen 443 ssl;
            listen 80;
            root /mnt;	
            ssl_certificate conf_ssl/1_snkefu.niuhome.com_bundle.crt;
            ssl_certificate_key conf_ssl/2_snkefu.niuhome.com.key;
            location / {
                #root   html;
                #index  index.html index.htm;
	            proxy_pass http://snkefu-web;
        	    proxy_redirect off;
        	    proxy_set_header HOST $host;
        	    proxy_set_header X-Real-IP $remote_addr;
        	    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        	    proxy_set_header Host $host;
                }
                error_page   500 502 503 504  /50x.html;
                location = /50x.html {
                    root   html;
                }
            }
        }

>需要配置的部分:server{}和http{}(下面只写要改的部分)  
server{  
    server_name snkefu.niuhome.com;  #域名  
    location/{  
        proxy_pass http://snkefu-web; #集群的配置  
    }  
}  
http {  
    upstream snkefu-web{  #upstream后的内容必须和proxy_pass对应
        server 218.244.149.110:8080; #两个服务器的地址,这就是传说中的集群  
        server 218.244.149.110:9081;  
        session_sticky; #解决session复制  
}  
######原理
浏览器访问snkefu.niuhome.com,被nginx拦截,nginx查看location的proxy_pass,proxy_pass决定打到那个地址.proxy_pass寻找配置文件里的snkefu-web地址,最终完成访问.nginx的集群默认是轮询算法.






