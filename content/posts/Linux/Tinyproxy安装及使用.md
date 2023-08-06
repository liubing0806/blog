
---
title: "Tinyproxy安装及使用"
date: 2023-08-05
tags: ["linux"]
---

``` bash
wget https://github.com/tinyproxy/tinyproxy/releases/download/1.10.0/tinyproxy-1.10.0.tar.gz

tar -zxvf tinyproxy-1.10.0.tar.gz

cd tinyproxy-1.10.0

# 编译安装
./configure && 
make && 
make install

which tinyproxy
/usr/local/bin/tinyproxy

tinyproxy -v
tinyproxy 1.10.0
```

``` bash
/etc/tinyproxy/tinyproxy.conf
```

``` bash
# 注释掉这行，允许所有ip访问
# Allow 127.0.0.1

# 权限校验
BasicAuth user 123456
```
指定配置文件启动
``` bash
# 启动（不采用后台启动，方便调试）
tinyproxy -d -c /etc/tinyproxy/tinyproxy.conf

# 指定配置文件启动（后台启动）
tinyproxy -c /etc/tinyproxy/tinyproxy.conf

# 杀掉进程
ps -ef|grep tinyproxy|grep -v grep|awk '{print "kill -9 "$2}'|sh

```


验证:
``` bash
# 不加验证参数不会正常返回
curl -x http://127.0.0.1:8888 www.baidu.com
Proxy Authentication Required

# 正常返回
curl -x http://user:123456@127.0.0.1:8888 www.baidu.com
```
