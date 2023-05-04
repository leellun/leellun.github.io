---
title: nginx基础知识
date: 2020-01-04 12:14:02
categories:
  - 服务器
  - nginx
tags:
  - nginx
---

# 1 开启日志打开缓存

对于每一条日志记录,日志文件都将先打开文件,再写入日志记录,然后马上关闭,为了提高包含变量的日志文件存放路径的性能,可以使用open_log_file_cache指令来设置,格式如下：

```
open_log_file_cache max=N [inactive=time][min_uses=N][valid=time]|off
```

该指令默认是禁止的,等同于：open_log_file_cache off;

max:设置缓存中的最大文件描述符数量

inactive:设置一个时间,如果在设置的时间内没有使用此文件描述符,则自动删除此描述符

min_uses:在参数inactive指定的时间范围内,如果日志文件超过被使用的次数,则将该日志文件的描述符计入缓存,默认为10秒钟

valid:设置多长时间检查一次,看日志文件路径与文件名是否仍然存在默认60秒

```
open_log_file_cache max=1000 inactive=20s min_uses=2 valid=1m;
```

# 2 Nginx开启压缩输出

在http {}中间,启用压缩

```
$ vim /etc/nginx/nginx.conf
gzip on;
gzip_min_length 1k;
gzip_buffer 4 16k;
gzip_http_version 1.1;
gzip_comp_level 2;
gzip_type text/plain application/x-javascript text/css application/xml;
gzip_vary on;
```

# 3 缓存

## 3.1 Nginx的浏览器本地缓存设置

```
$ vim /etc/nginx/nginx.conf
location ~.*\.(gif|jpg|jpge|png|bmp|swf)$
{
  expires 30d;
}
location ~.*\.(js|css)?$
{
  expires 1h;
}
```

## 3.2 nginx代理缓存

https://blog.csdn.net/weixin_30795127/article/details/97385091#proxy_cache_path

```
location ~.*\.(gif|jpg|jpge|png|bmp|swf|js|css|html)$
{
  proxy_cache cache_one;
  proxy_cache_valid 200 10m;
  proxy_cache_valid 304 1m;
  proxy_cache_valid 301 302 1h;
  proxy_cache_valid any 1m;
  #以域名、URI、参数组合成Web缓存的Key值, Nginx根据Key值哈希,存储缓存内容到二级缓存目录内
  proxy_cache_key $hostSuri$is_argsSargs;
  proxy_set_header Host $host;
  proxy_set_header X-Forwarded-For $remote_addr;
  proxy_pass http://192.168.3.1;
}
```

proxy_cache_valid

```
 语法: proxy_cache_valid [code ...] time; 
```

# 4 设定限速

```
location /download
{
  limit_rate 256k;
  proxy_pass http://1.2.3.4;
  proxy_set_header Host $host;
  proxy_set_header X-Forwarded-For $remote_addr;
}
location /movie
{
  limit_rate_after 10m;
  limit_rate 100k;
}
if (Shttp_user_agent ~ Google|Yahoo|baidu){
  limit_rate 20k;
}
```

# 5 请求连接控制

允许客户端请求的最大的单个文件字节数

client_max_body_size300m;

缓冲区代理缓冲用户端请求的最大字节数可以理解为先保存到本地再传给用户

client_body_buffer_size 128k;

跟后端服务器连接的超时时间-发起握手等候响应超时时间

proxy_connect_timeout 600;

连接成功后-等候后端服务器响应时间_其实已经进入后端的排队之中等候处理

proxy_read_timeout600

后端服务器数据回传时间-就是在规定时间之内后端服务器必须传完所有的数据

proxy_send_timeout 600;

代理请求缓存区_这个缓存区间会保存用户的头信息以供Nginx进行规则处理-一般只要能保存下头信息即可

proxy_buffer_size 16k;

# 6 Nginx rewrite

Rewrite主要的功能就是实现URL的重写, Nginx的Rewrite规则采用PCRE (Perl Compatible Regular Expressions) Perl兼容正则表达式的语法进行规则匹配,如果需要Nginx的Rewrite功能,在编译Nginx之前,需要编译安装PCRE库

URL是Uniform Resource Location的缩写,译为“统一资源定位符”。如: http://www.linkwan.com/111/welcome.htm

URI由一个通过通用资源标志符(Universal ResourceIdentifier, 简称"URI")进行定位

# 7 指令

if指令

规则语法

```
if ($http_user_agent ~ MSIE ){
  rewrite ^(.*)$ /msie/$1 break;
}
if (!-f $request_filename){
  rewrite ^/img/(.*)$/site/$host/images/$1 last;
}
```

return指令

示例,如果访问的URL以".sh"、"*.bash"结尾,则返回状态码403

```
location .*.(sh | bash)?$
return 403;
```

set、rewrite指令

```
if (?host ~* ^(.*?)\.aaa\.com$)
{
  set ?var_tz '1';
  if (?host ~* ^192\.168\.1\.(.*?)$)
  {
      set $var_tz '1';
  }
  if ($host ~* ^localhost)
  {
      set ?var_tz '1'
  }
  if ($var_tz !~'1')
  {
      rewrite ^/(.*)$ http://www.aaa.com/ redirect;
  }
```

rewrite 

```
location /cms/ [
  proxy_pass http://test.yourdomain.com;
  rewrite "^/cms/(.*)\.html$" /cms/index.html break;
}
一般在跟location中(即location/[..])或直接在server标签中编写rewrite规则,推荐使用last标记,在非跟location中(location /cms/{...}),则使用break标记
```

rewrite规则编写实例

1、将原来要访问/data目录重写为/bbs

```
rewrite A/data/?$ /bbs/ permanent;
```

2、根据不同的浏览器将得到不同的结果

```
if ($http_user_agent MSIE){
  rewrite ^(.*)$ /msie/$1 break;
}



```

3、防止盗链

```
location ~*\.(gif|jpg|png|swf|flv)$ {
  valid_referers none blocked www.test.com *.test.com;
  if ($invalid_referer) (
     rewrite ^/(.*) http://www.test.com/block.html;
  }
}
```

4、实现域名跳转

所有对www.abc.com的访问, redirect到www.test.com

```
server
{
listen 80;
server_name www.test.com;
index index.html index.php;
root /export/home/www
if ($host ="www.abc.com"){
rewrite ^/(.*)$ http://www.test.com/$1 permanent;
}
}
```



# 8 目录身份验证

```
# htpasswd-cm /etc/nginx/.htpasswd alice
# htpasswd-m /etc/nginx/.htpasswd bob
location /redhat {
root /web/html;
index index.html index.htm;
autoindex on;
auth_basic "AwstatAuth"
auth_basic_user_file /etc/nginx/.htpasswd;
deny 192.168.0.132;
allow 192.168.0.0/24;
allow 192.168.1.1;
deny all;
}

```

# 9 图片裁剪

要实现nginx的图片裁剪功能，需要在编译nginx时加入第三方模块ngx_http_image_filter_module，该模块提供了图片处理功能，包括裁剪、缩放、旋转等。 

```
./configure --with-http_image_filter_module
```

```
http {
    ...
    # 在http模块中添加 image_filter 指令
    image_filter        on;
    # 添加 image_filter_jpeg_quality 指令，指定jpeg图片压缩质量
    image_filter_jpeg_quality 75;
    # 添加 image_filter_buffer 指令，指定缓存大小
    image_filter_buffer 2M;
    ...
}

```

# 10 常用命令

1. 启动Nginx：`sudo nginx`
2. 停止Nginx：`sudo nginx -s stop`
3. 重载Nginx配置文件：`sudo nginx -s reload`
4. 重新启动Nginx：`sudo nginx -s reopen`
5. 检查Nginx配置文件语法：`sudo nginx -t`
6. 查看Nginx版本号：`sudo nginx -v`
7. 查看Nginx的编译配置选项：`sudo nginx -V`
8. 查看Nginx当前连接数：`sudo nginx -V`
9. nginx -c nginx.conf 指定配置文件启动