## LiveHouse 反向代理（NGINX 版）

NGINX 版 Livehouse 反代用于解决内地访问 Livehouse 无法进入直播间问题。

### 如何部署

修改 NGINX 的虚拟主机配置文件如下保存即可，注意修改相关参数，NGINX 必须启用 SUB FILTER 模块！

```
geo $dollar {
    default "$";
}

server {
    listen            <绑定IP>:<端口>;
    server_name       <绑定域名>;

    # avoid caching by proxies
    add_header        Cache-Control public;
    
    # Handle CDN Resources
    location /cdn-dat {
        root <WWWROOT目录>;
        expires 7d;
    }
    
    # Handle Advertisement Fake URI
    location /cdn-adb {
        return 200;
        expires 36500d;
    }

	# Handle subdomain URI
    location /cdn-sub/record {
        resolver 8.8.8.8;
        rewrite ^/cdn-sub/record/(.*)$ /$1 break;
        proxy_pass https://record.livehouse.in/;
    }
    location /cdn-sub/event {
        resolver 8.8.8.8;
        rewrite ^/cdn-sub/event/(.*)$ /$1 break;
        proxy_pass https://event.livehouse.in/;
    }

    # Handle any other URI
    location / {
        resolver 8.8.8.8;
        proxy_pass https://livehouse.in/;
        sub_filter_once off;
 
        # Proxy Settings
        proxy_redirect     off;
        proxy_set_header   Host             'livehouse.in';
        proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        proxy_set_header   X-Cache-Server   'RazerNiz Cache v1.0.0';
        proxy_set_header   Accept-Encoding  '';
        proxy_cookie_domain livehouse.in $host;
        proxy_ignore_headers X-Accel-Expires Expires Cache-Control Set-Cookie; 
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
        proxy_max_temp_file_size   0;
        proxy_connect_timeout      90;
        proxy_send_timeout         90;
        proxy_read_timeout         90;
        proxy_buffer_size          4k;
        proxy_buffers              4 32k;
        proxy_busy_buffers_size    64k;
        proxy_temp_file_write_size 64k;
        
        # Replacing Resource blocked in China
        sub_filter 'https://ajax.googleapis.com/ajax/libs/jquery/' '//cdn.bootcss.com/jquery/';
        sub_filter 'https://ajax.googleapis.com/ajax/libs/angularjs/' '//cdn.bootcss.com/angular.js/';
        sub_filter 'https://cdn.firebase.com/js/client/1.1.2/firebase.js' '/cdn-dat/js/firebase.js';
        sub_filter '//livehouse.in/' '//$host/';
        sub_filter '//record.livehouse.in/' '//$host/cdn-sub/record/';
        sub_filter '//event.livehouse.in/' '//$host/cdn-sub/event/';
        
        # Replacing livehouse.in Flash Player
        sub_filter '//static.cdn.livehouse.in/assets/GrindPlayer-4a087377ed.swf' '//static.cdn.livehouse.in/assets/video-js-flashls-194ba172ae.swf';
        sub_filter 'grind' 'flashls';
        
        # Advertisements Filter
        sub_filter '//imasdk.googleapis.com/' '/cdn-adb/';
        sub_filter '//pagead2.googlesyndication.com/' '/cdn-adb/';
        sub_filter '//partner.googleadservices.com/' '/cdn-adb/';
        sub_filter '//www.googletagservices.com/' '/cdn-adb/';
        sub_filter '//www.google-analytics.com/' '/cdn-adb/';
        sub_filter 'https://connect.facebook.net/' '/cdn-adb/';
        sub_filter '//connect.facebook.net/' '/cdn-adb/';
        
        # Adding T/S Chinese Convernt
        sub_filter '</body>' '<script src="/cdn-dat/js/jquery.s2t.js" type="text/javascript"></script><script type="text/javascript">$dollar(\'body\').t2s();</script></body>';
        sub_filter '</title>' ' - 简体中文镜像</title>';
    }
}
```

### 更新内容

1. 对 Google Firebase 库单独本地缓存
2. 强行移除多余 Javascript 脚本，例如 Facebook SDK、广告等
3. 添加页面繁体中文转简体中文 JS 库，默认启用
4. Web 服务器添加 POST 与 Cookies 支持

### 已知问题

1. 无法删除重播录像