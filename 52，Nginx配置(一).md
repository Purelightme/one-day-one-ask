### ngx_http_core_module

alias path;// 如果需要修改 uri，使用它，否则用 root 。它将 location 匹配后剩余的部分作为 uri 。

client_body_in_file_only on; // 主要用于调试。

client_body_temp_path /var/log/nginx/client_body_temp 1 2;

client_max_body_size 1m; // request 超过该大小，返回 413 Entity Too Large 。

default_type application/octet-stream;

types {

​    text/html json;

}

error_page 404 /4xx.html;  // 根据 status code 返回指定内容。

etag on;

if_modified_since before; // 资源 response header 会增加 Last-Modified 指定资源最后修改时间，request hedaer 包含 If-Modified-Since ，并且该资源在这时间之后无修改，则返回 304 Not Modified 状态码，响应体为空，常用于浏览器，可以节约带宽资源，提升响应速度。

keepalive_requests number;

keepalive_time

keepalive_timeout

limit_except GET {

​    deny all;

} // 除了 GET ，其他方法访问一律拒绝，返回 403 Forbidden 。

limit_rate 1k; // 这将限制 server 每秒向客户端最多发送 1k 的速率。谨慎使用。

lingering_close on; // instructs nginx to wait for and process additional data from a client before fully closing a connection, but only if heuristics suggests that a client may be sending more data.

listen unix:/var/log/nginx/nginx.sock; // 会影响普通 ip:port 的服务，curl --unix-socket ./nginx.sock -X POST http://ng.com/post

log_not_found on; // 404 写入到 error_log 。

merge_slashes on;

root path;

server {
    server_name ~^(www\.)?(.+)$; // 支持正则表达式

​    location / {
​        root /sites/$2;
​    }

} 

server_tokens off; // 隐藏 Nginx 版本号。

`try_files $uri $uri/ /c/default.txt =405;` // $uri/ 检查目录是否存在，按顺序返回第一个存在的文件。

变量：

```nginx
location /d {
        return 200 "age:$arg_age,
        binary_remote_addr:$binary_remote_addr,
        body_bytes_sent:$body_bytes_sent,
        bytes_sent:$bytes_sent,
        connection:$connection,
        connection_requests:$connection_requests,
        connection_time:$connection_time,
        content_length:$content_length,
        content_type:$content_type,
        cookie_id:$cookie_id,
        document_root:$document_root,
        document_uri:$document_uri,
        host:$host,
        hostname:$hostname,
        http_a:$http_a,
        https:$https,
        is_args:$is_args,
        limit_rate:$limit_rate,
        msec:$msec,
        nginx_version:$nginx_version,
        pid:$pid,
        pipe:$pipe,
        proxy_protocol_addr:$proxy_protocol_addr,
        query_string:$query_string,
        realpath_root:$realpath_root,
        remote_addr:$remote_addr,
        remote_port:$remote_port,
        remote_user:$remote_user,
        request:$request,
        request_body:$request_body,
        request_completion:$request_completion,
        request_filename:$request_filename,
        request_id:$request_id,
        request_length:$request_length,
        request_method:$request_method,
        request_time:$request_time,
        request_uri:$request_uri,
        scheme:$scheme,
        server_addr:$server_addr,
        server_name:$server_name,
        server_port:$server_port,
        server_protocol:$server_protocol,
        status:$status,
        time_iso8601:$time_iso8601,
        time_local:$time_local,
        uri:$uri,";
    }
```

### ngx_http_access_module

allow , deny 。控制

### ngx_http_addition_module

```nginx
location /e {
        add_before_body /a/a1.txt;
        add_after_body /a/a1.txt;
        addition_types *;
    }
```

功能是在 body 前后追加内容。

### ngx_http_autoindex_module

```nginx
location / {
        autoindex on;
        autoindex_format json; //以 json 格式输出
    }
```

### ngx_http_flv_module

```nginx
 location /flv {
        flv;
    }
```

支持 flv 视频播放。

### ngx_http_headers_module

```nginx
add_header X-PURE light always; // 不加 always 仅针对特定 status code 。默认会覆盖其他 context 的配置。
```

### ngx_http_mp4_module

```nginx
location /video {
        sendfile on;
        #mp4;
    }
```

支持 mp4 文件的 start ,end 参数控制视频时间。

### ngx_http_log_module

```nginx
location / {
	if ($time_iso8601 ~ "^(\d{4})-(\d{2})-(\d{2})T(\d{2}):(\d{2}):(\d{2})") {
            set $year $1;
            set $month $2;
            set $day $3;
            set $hour $4;
            set $minutes $5;
            set $seconds $6;
    }
    
    access_log /var/log/nginx/ngcom.access.log jlog;
}

http {
	log_format jlog  '{\"method\":\"$request_method\",\"uri\":\"$request_uri\",\"query_string\":\"$query_string\",\"remote_addr\":\"$remote_addr\",\"scheme\":\"$scheme\",\"content_length\":$content_length,\"time_local\":\"$time_local\",\"request_id\":\"$request_id\",\"bytes_sent\":$bytes_sent,\"status\":$status,\"time\":\"$year-$month-$day $hour:$minutes:$seconds\",\"request_time\":$request_time}';
}

// 效果：{"method":"GET","uri":"/c/c1.json","query_string":"-","remote_addr":"172.18.0.1","scheme":"http","content_length":-,"time_local":"10/Mar/2022:10:43:24 +0800","request_id":"ae8075fbd6c29847eada920635be3fb4","bytes_sent":440,"status":200,"time":"2022-03-10 10:43:24","request_time":0.000}
```

Docker Nginx 容器添加环境变量 TZ: Asia/Shanghai ，解决时间慢8小时的问题。

### ngx_http_map_module

```nginx
http {
	map $http_user_agent $is_postman {
        default 0;
        "~Postman" 1;
  }
}

location / {
	add_header POSTMAN $is_postman always;
}
```

用已知变量定义新变量。

### ngx_http_realip_module

```nginx
set_real_ip_from  192.168.1.0/24;
set_real_ip_from  192.168.2.1;
set_real_ip_from  2001:0db8::/32;
real_ip_header    X-Forwarded-For;
real_ip_recursive on;
```

### ngx_http_referer_module

```nginx
location /g {
    valid_referers none server_names *.example.com;
    if ($invalid_referer) {
        return 403;
    }
    return 200 "ok";
}
```

主要作用是防盗链。

### ngx_http_rewrite_module

```nginx
rewrite_log on;
rewrite ^/g/(.*)$ http://ng.com:81/$1;
// flags: 
last 继续匹配剩余 location 
break 不再匹配其他 location
redirect 302 发起两次请求
permanent 301
```

### ngx_http_secure_link_module

```nginx
MacOS 生成MD5 ：echo -n '1646907464/h/h1.txt172.18.0.1 secret' | \
    openssl md5 -binary | openssl base64 | tr +/ -_ | tr -d =
Nginx 配置：
location /h {
        secure_link $arg_md5,$arg_expires;
        secure_link_md5 "$secure_link_expires$uri$remote_addr secret";
        if ($secure_link = "") {
            return 403;
        }
        if ($secure_link = "0") {
            return 410;
        }
        return 200 "hhhhhhhhhh";
    }
```

