### Server字段是什么？

![7.1](https://raw.githubusercontent.com/Purelightme/one-day-one-ask/master/images/7.1.png)

像这样的，如果使用过nginx，apache等等软件的都见过这个字段，这个一般是服务器软件写死在源码里面的，不好直接更改。但是不更改的话，别人一看就知道你用的什么软件，像图中这个，连服务器操作系统都暴露出来了，非常的不安全。

### 如何更改？(nginx举例)

- 直接修改源码，重新编译。
- 使用第三方模块[headers-more-nginx-module](https://github.com/openresty/headers-more-nginx-module) 

### 其他选择

> 今天尝试下使用[openresty](http://openresty.org/cn/getting-started.html)
>
> openresty是nginx+lua的结合版本，更强大，已经内置了headers-more-nginx-module这个模块。

#### 实验流程

1. 安装

   ```
   brew install openresty/brew/openresty
   ```

2. 配置文件

   ```nginx
   worker_processes  2;
   error_log logs/error.log;
   events {
       worker_connections 1024;
   }
   http {
       server {
           listen 8080;
   	      root html;
       		more_set_headers "Server: Purelightme/1.0";
           location / {
               default_type text/html;
               content_by_lua_block {
                   ngx.say("<p>hello, world</p>")
               }
           }
   				location ~ \.php$ {
   	    			try_files $uri /index.php =404;
               fastcgi_pass 127.0.0.1:9000;
               fastcgi_index index.php;
               fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
               include fastcgi_params;
   				}
       }
   }
   ```

   相关文件目录：

   ![7.3](https://raw.githubusercontent.com/Purelightme/one-day-one-ask/master/images/7.3.png)

3. 运行

   ```
   cd ~/Desktop/openresty_env
   /usr/local/Cellar/openresty/1.15.8.1/nginx/sbin/nginx -p `pwd`/  -c conf/nginx.conf
   ```

4. 效果

   ![7.2](https://raw.githubusercontent.com/Purelightme/one-day-one-ask/master/images/7.2.png)

### 最后

还有一个 ”X-Powered-By“ 这个响应头也是经常可以看到，这个不是服务器软件输出的，而是应用层设置的，就拿PHP来说，php.ini里面有一个与之相关的配置项：expose_php，默认是On，效果就是响应头有”PHP/7.2.13“类似的东西，也是不安全的，改成Off，重启服务器软件(nginx)就可以删除了。

```2019-07-19```