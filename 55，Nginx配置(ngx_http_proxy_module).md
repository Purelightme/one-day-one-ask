```
Directives
     proxy_bind
     proxy_buffer_size
     proxy_buffering
     proxy_buffers
     proxy_busy_buffers_size
     proxy_cache
     proxy_cache_background_update
     proxy_cache_bypass
     proxy_cache_convert_head
     proxy_cache_key
     proxy_cache_lock
     proxy_cache_lock_age
     proxy_cache_lock_timeout
     proxy_cache_max_range_offset
     proxy_cache_methods
     proxy_cache_min_uses
     proxy_cache_path
     proxy_cache_purge
     proxy_cache_revalidate
     proxy_cache_use_stale
     proxy_cache_valid
     proxy_connect_timeout
     proxy_cookie_domain
     proxy_cookie_flags
     proxy_cookie_path
     proxy_force_ranges
     proxy_headers_hash_bucket_size
     proxy_headers_hash_max_size
     proxy_hide_header
     proxy_http_version
     proxy_ignore_client_abort
     proxy_ignore_headers
     proxy_intercept_errors
     proxy_limit_rate
     proxy_max_temp_file_size
     proxy_method
     proxy_next_upstream
     proxy_next_upstream_timeout
     proxy_next_upstream_tries
     proxy_no_cache
     proxy_pass
     proxy_pass_header
     proxy_pass_request_body
     proxy_pass_request_headers
     proxy_read_timeout
     proxy_redirect
     proxy_request_buffering
     proxy_send_lowat
     proxy_send_timeout
     proxy_set_body
     proxy_set_header
     proxy_socket_keepalive
     proxy_ssl_certificate
     proxy_ssl_certificate_key
     proxy_ssl_ciphers
     proxy_ssl_conf_command
     proxy_ssl_crl
     proxy_ssl_name
     proxy_ssl_password_file
     proxy_ssl_protocols
     proxy_ssl_server_name
     proxy_ssl_session_reuse
     proxy_ssl_trusted_certificate
     proxy_ssl_verify
     proxy_ssl_verify_depth
     proxy_store
     proxy_store_access
     proxy_temp_file_write_size
     proxy_temp_path
Embedded Variables
```

大体上跟 fastcgi module 差不多，这里主要关注差异点。

##### proxy_pass uri;

示例：proxy_pass http://host:port/uri;

- 不带uri，不带 "/"

  ```nginx
  location /proxy {
      proxy_pass http://nginx2;
      proxy_http_version 1.1;
  }
  
  location / {
      return 200 "request_uri:$request_uri";
  }
  ```

  访问 http://ng.com/proxy/aaa?id=1 ，得到结果：request_uri:/proxy/aaa?id=1

- 不带 uri ，带 ”/"

  ```nginx
  location /proxy {
      proxy_pass http://nginx2/;
      proxy_http_version 1.1;
  }
  
  location / {
      return 200 "request_uri:$request_uri";
  }
  ```

  访问 http://ng.com/proxy/aaa?id=1 ，得到结果：request_uri://aaa?id=1

- 带 uri ，不带 "/"

  ```nginx
  location /proxy {
      proxy_pass http://nginx2/xxx;
      proxy_http_version 1.1;
  }
  
  location / {
      return 200 "request_uri:$request_uri";
  }
  ```

  访问 http://ng.com/proxy/aaa?id=1 ，得到结果：request_uri:/xxx/aaa?id=1

- 带 uri ，带 "/"

  ```nginx
  location /proxy {
      proxy_pass http://nginx2/xxx/;
      proxy_http_version 1.1;
  }
  
  location / {
      return 200 "request_uri:$request_uri";
  }
  ```

  访问 http://ng.com/proxy/aaa?id=1 ，得到结果：request_uri:/xxx//aaa?id=1

- rewrite 情况

  ```nginx
  location /proxy {
      rewrite  /users/([^/]+) /users?id=$1 break;
      proxy_pass http://nginx2/xxx/;
      proxy_http_version 1.1;
  }
  
  location / {
      return 200 "request_uri:$request_uri";
  }
  ```

  访问 http://ng.com/proxy/users/20 ，得到结果：request_uri:/users?id=20

  携带的 uri 直接被忽略了。

- 使用变量

  ```nginx
  location /proxy {
      proxy_pass http://nginx2$request_uri;
      proxy_http_version 1.1;
  }
  
  location / {
      return 200 "request_uri:$request_uri";
  }
  ```

  访问 http://ng.com/proxy/aaa?id=20 ，得到结果：request_uri:/proxy/aaa?id=20

##### 修改请求内容

proxy_method，proxy_pass_request_headers，proxy_set_header，proxy_pass_request_body，proxy_set_body，这几个指令可以修改请求内容。

proxy server 后端代码：

```php
<?php
header('Content-Type:application/json');
echo json_encode([
	'uri'=> $_SERVER['REQUEST_URI'],
	'method' => $_SERVER['REQUEST_METHOD'],
	'body' => file_get_contents("php://input")
]);
```

请求：http://ng.com/proxy/xxx 不带任何参数。

```nginx
location /proxy {
    proxy_pass http://nginx2$request_uri;
    proxy_http_version 1.1;
}
```

该配置下返回：

```json
{
    "uri": "/proxy/xxx",
    "method": "GET",
    "body": ""
}
```

使用新配置：

```nginx
location /proxy {
    proxy_pass http://nginx2$request_uri;
    proxy_http_version 1.1;
    proxy_method POST;
    proxy_set_header Author purelight;
    proxy_set_body hello;
}

```

返回：

```json
{
    "uri": "/proxy/xxx",
    "method": "POST",
    "body": "hello"
}
```

##### proxy_redirect

修改重定向内容，后端 php 代码：

```php
<?php

header("Location:https://www.baidu.com");
```

访问 http://ng.com/proxy/xxx 默认返回status code = 200，使用浏览器会自动重定向：

```html
<html>

<head>
	<script>
		location.replace(location.href.replace("https://","http://"));
	</script>
</head>

<body>
	<noscript>
		<meta http-equiv="refresh" content="0;url=http://www.baidu.com/"></noscript>
</body>

</html>
```

添加配置：

```nginx
location /proxy {
    proxy_pass http://nginx2$request_uri;
    proxy_http_version 1.1;
    proxy_redirect https://www.baidu.com https://www.qq.com;
}
```

再次访问发现跳转到腾讯网了。

##### Websocket

基本上是固定写法：

```nginx
location /chat/ {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}
```

优化版：

```nginx
http {
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }

    server {
        ...

        location /chat/ {
            proxy_pass http://backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
        }
    }
}    
```

##### ssl

参考：ngx_http_ssl_module。

