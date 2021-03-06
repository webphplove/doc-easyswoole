# Swoole Http Server
swoole-1.7.7起增加了内置Http服务器的支持,swoole_http_server 继承自swoole_server，是一个完整的http服务器实现，通过几行代码即可写出一个异步非阻塞多进程的Http服务器。
```
$http = new swoole_http_server("127.0.0.1", 9501);
$http->on('request', function ($request, $response) {
    $response->end("<h1>Hello Swoole. #".rand(1000, 9999)."</h1>");
});
$http->start();
```
> swoole_http_server对Http协议的支持并不完整，建议仅作为应用服务器。并且在前端增加Nginx或者Apache作为代理,仅将API请求转发给Swoole Server处理。

### Nginx转发规则
```
server {
    root /data/wwwroot/;
    server_name local.swoole.com;
    location / {
        proxy_http_version 1.1;
        proxy_set_header Connection "keep-alive";
        proxy_set_header X-Real-IP $remote_addr;
        if (!-e $request_filename) {
             proxy_pass http://127.0.0.1:9501;
        }
    }
}
```
### Apache转发规则
```
<IfModule mod_rewrite.c>
  Options +FollowSymlinks
  RewriteEngine On
  RewriteCond %{REQUEST_FILENAME} !-d
  RewriteCond %{REQUEST_FILENAME} !-f
  #RewriteRule ^(.*)$ index.php/$1 [QSA,PT,L]  fcgi下无效
  RewriteRule ^(.*)$  http://127.0.0.1:9501/$1 [QSA,P,L]
   #请开启 proxy_mod proxy_http_mod request_mod
</IfModule>
```

# 常见问题
## CURL发送POST请求服务器端超时 
   CURL在发送较大的POST请求时会先发一个100-continue的请求，如果收到服务器的回应才会发送实际的POST数据。而swoole_http_server不支持100-continue，就会导致CURL请求超时。
   
   解决办法是关闭CURL的100-continue
   ```
   // 创建一个新cURL资源
   $ch = curl_init();
   // 设置URL和相应的选项
   curl_setopt($ch, CURLOPT_URL, "http://127.0.0.1:9501");
   curl_setopt($ch, CURLOPT_HEADER, 0);
   curl_setopt($ch, CURLOPT_POST, 1); //设置为POST方式
   curl_setopt($ch, CURLOPT_HTTPHEADER, array('Expect:'));
   curl_setopt($ch, CURLOPT_POSTFIELDS, array('test' => str_repeat('a', 800000)));//POST数据
   curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
   ```
### 其他客户端
   如果客户端是其他语言编写的，无法修改客户端去掉100-continue，那么还有2个方案可以解决此问题。
   - 使用Nginx做前端代理，由Nginx处理100-Continue
   - 重新编译Swoole启用100-Continue的支持，需要手工修改swoole_config.h，找到SW_HTTP_100_CONTINUE，去掉注释，执行make clean && make install
   
    > 启用100-CONTINUE后会额外消耗服务器的CPU资源
    
## 使用Chrome/FireFox访问服务器会产生2次请求
   这是因为Chrome浏览器会自动请求一次favicon.ico，所以服务器会收到2个Request，通过打印$req->server['request_uri'] 能看到请求的URL路径。
   
## GET/POST请求的最大尺寸 
### GET请求最大8192
   GET请求只有一个Http头，swoole底层使用固定大小的内存缓存区8K，并且不可修改。如果请求不是正确的Http请求，将会出现错误。底层会抛出以下错误：
   ```
   WARN swReactorThread_onReceive_http_request: http header is too long.
   ```
### POST/文件上传
   最大尺寸受到 package_max_length 配置项限制，默认为2M，可以调用swoole_server->set传入新的值修改尺寸。swoole底层是全内存的，因此如果设置过大可能会导致大量并发请求将服务器资源耗尽。 计算方法：最大内存占用 = 最大并发请求数 * package_max_length
   
<script>
    var _hmt = _hmt || [];
    (function() {
        var hm = document.createElement("script");
        hm.src = "https://hm.baidu.com/hm.js?4c8d895ff3b25bddb6fa4185c8651cc3";
        var s = document.getElementsByTagName("script")[0];
        s.parentNode.insertBefore(hm, s);
    })();
</script>   
<script>
(function(){
    var bp = document.createElement('script');
    var curProtocol = window.location.protocol.split(':')[0];
    if (curProtocol === 'https') {
        bp.src = 'https://zz.bdstatic.com/linksubmit/push.js';        
    }
    else {
        bp.src = 'http://push.zhanzhang.baidu.com/push.js';
    }
    var s = document.getElementsByTagName("script")[0];
    s.parentNode.insertBefore(bp, s);
})();
</script>
