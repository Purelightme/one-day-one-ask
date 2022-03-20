```
Directives
     fastcgi_bind
     fastcgi_buffer_size
     fastcgi_buffering
     fastcgi_buffers
     fastcgi_busy_buffers_size
     fastcgi_cache
     fastcgi_cache_background_update
     fastcgi_cache_bypass
     fastcgi_cache_key
     fastcgi_cache_lock
     fastcgi_cache_lock_age
     fastcgi_cache_lock_timeout
     fastcgi_cache_max_range_offset
     fastcgi_cache_methods
     fastcgi_cache_min_uses
     fastcgi_cache_path
     fastcgi_cache_purge
     fastcgi_cache_revalidate
     fastcgi_cache_use_stale
     fastcgi_cache_valid
     fastcgi_catch_stderr
     fastcgi_connect_timeout
     fastcgi_force_ranges
     fastcgi_hide_header
     fastcgi_ignore_client_abort
     fastcgi_ignore_headers
     fastcgi_index
     fastcgi_intercept_errors
     fastcgi_keep_conn
     fastcgi_limit_rate
     fastcgi_max_temp_file_size
     fastcgi_next_upstream
     fastcgi_next_upstream_timeout
     fastcgi_next_upstream_tries
     fastcgi_no_cache
     fastcgi_param
     fastcgi_pass
     fastcgi_pass_header
     fastcgi_pass_request_body
     fastcgi_pass_request_headers
     fastcgi_read_timeout
     fastcgi_request_buffering
     fastcgi_send_lowat
     fastcgi_send_timeout
     fastcgi_socket_keepalive
     fastcgi_split_path_info
     fastcgi_store
     fastcgi_store_access
     fastcgi_temp_file_write_size
     fastcgi_temp_path
Parameters Passed to a FastCGI Server
Embedded Variables
```

##### fastcgi_buffer 相关：

```nginx
location ~ \.php$ {
    fastcgi_pass fpm;
    fastcgi_buffer_size 4k;
    fastcgi_buffers 8 4k;
    fastcgi_buffering on; # 开启 buffer
    fastcgi_max_temp_file_size 1m;
    fastcgi_temp_path /var/log/nginx/fastcgi.temp 1;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    include fastcgi_params;
}
```

fastcgi_buffer_size 定义读取 fastcgi 应答第一部分需要多大缓冲区，该值表示使用1个4kb的缓冲区读取应答第

一部分(应答头),可以设置为 fastcgi_buffers 选项缓冲区大小。

所以 buffer 一共是 8 * 4k + 4k 大小。超出部分将会用临时文件存储，请求结束删除。

```nginx
client_body_buffer_size 1024k;
client_max_body_size 2050m;
```

这是针对客户端设置的 buffer 。

##### fastcgi_cache 相关：

```nginx
location ~ \.php$ {
        #fastcgi_pass fpm;
        #fastcgi_buffer_size 4k;
        #fastcgi_buffers 8 4k;
        #fastcgi_buffering on;
        #fastcgi_max_temp_file_size 1m;
        #fastcgi_temp_path /var/log/nginx/fastcgi.temp 1;
        #fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        #include fastcgi_params;
  			# cache :
        fastcgi_cache zzz;
        fastcgi_cache_min_uses 1;
        fastcgi_ignore_headers Cache-Control Expires Set-Cookie;
        fastcgi_cache_key $request_uri;
        fastcgi_cache_valid 200 1m;
        add_header X-Cache $upstream_cache_status;
}
```

fastcgi_cache_path 指定缓存路径，缓存内容：

```shell
$ cat nginx/log/fastcgi.cache/1/af/e251273eb74a8ee3f661a7af00915af1
,z0b���������y0b��"oa�
KEY: /index.php
=Content-type: text/html; charset=UTF-8

2022-03-15 19:35:12%
```

fastcgi_hide_header Author; 隐藏某些 header 。

fastcgi_ignore_client_abort  on; 

> Determines whether the connection with a FastCGI server should be closed when a client closes the connection without waiting for a response.

fastcgi_intercept_errors off;

> response status >= 300 将会交给 error_page 指令，当做错误处理。

fastcgi_keep_conn on;

> 建议开启，即 keepalive 。

fastcgi_next_upstream `error` | `timeout` | `invalid_header` | `http_500` | `http_503` | `http_403` | `http_404` | `http_429` | `non_idempotent` | `off` ...;

> 指示是否交个下一个上游，即判断本次 fastcgi 请求是否失败（unsuccessful attempts ）。
>
> 常见配置有 error ,timeout 超时，http_5xx 状态码，non_idempotent 非POST请求，off 禁用，即全部视为成功。

fastcgi_next_upstream_tries 最大尝试次数。

fastcgi_no_cache string;

> 至少一个非空非0，就不cache。示例：
>
> > ```
> > fastcgi_no_cache $cookie_nocache $arg_nocache$arg_comment;
> > fastcgi_no_cache $http_pragma    $http_authorization;
> > ```

特别的，对于 php ：

```
The following example shows the minimum required settings for PHP:

fastcgi_param SCRIPT_FILENAME /home/www/scripts/php$fastcgi_script_name;
fastcgi_param QUERY_STRING    $query_string;
The SCRIPT_FILENAME parameter is used in PHP for determining the script name, and the QUERY_STRING parameter is used to pass request parameters.

For scripts that process POST requests, the following three parameters are also required:

fastcgi_param REQUEST_METHOD  $request_method;
fastcgi_param CONTENT_TYPE    $content_type;
fastcgi_param CONTENT_LENGTH  $content_length
```

fastcgi_pass_request_body，fastcgi_pass_request_headers 控制是否把客户端的 body，header 传给 fastcgi server ，一般不使用。

3个 timeout :

```
fastcgi_connect_timeout 10s;
fastcgi_read_timeout 10s;
fastcgi_send_timeout 10s;
```

fastcgi_split_path_info 示例：

```nginx
location ~ ^(.+\.php)(.*)$ {
    fastcgi_split_path_info       ^(.+\.php)(.*)$;
    fastcgi_param SCRIPT_FILENAME /path/to/php$fastcgi_script_name;
    fastcgi_param PATH_INFO       $fastcgi_path_info;
}
```

> for the “`/show.php/article/0001`” request, the `SCRIPT_FILENAME` parameter will be equal to “`/path/to/php/show.php`”, and the `PATH_INFO` parameter will be equal to “`/article/0001`”.

