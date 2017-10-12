---
title: nodejs反向代理
date: 2017-10-13 06:13:12
tags: 反向代理
categories: nodejs
---
正向代理代理客户端，反向代理代理服务器。
使用nodejs实现简单的反向代理需要
1、创建一个http服务
2、服务接收到客户端请求后，去请求目的url
3、获取请求结果，返回给客户端


```
var http = require('http');
var https = require('https');
var path = require('path');

console.log('run')
// Create HTTP Server
http.createServer(function(req, res) {

    var uri = 'https://github.com/' + req.url;
    var file = req.url.replace(/\?.*/ig, '');
    var ext = path.extname(file);
    var type = getContentType(ext);

    res.writeHead(200, {
        'Content-Type': type
    });

    https.get(uri, function(response) {
        response.addListener('data', function(chunk)
        {
            res.write(chunk, 'binary');
        });
        response.addListener('end', function()
        {
            res.end();
        });
    });
}).listen(3000);

// Get content-type
var getContentType = function(ext) {
    var contentType = '';
    switch (ext) {
        case "":
            contentType = "text/html";
            break;
        case ".html":
            contentType = "text/html";
            break;
        case ".js":
            contentType = "text/javascriptss";
            break;
        case ".css":
            contentType = "text/css";
            break;
        case ".gif":
            contentType = "image/gif";
            break;
        case ".jpg":
            contentType = "image/jpeg";
            break;
        case ".png":
            contentType = "image/png";
            break;
        case ".ico":
            contentType = "image/icon";
            break;
        default:
            contentType = "application/octet-stream";
    }

    return contentType;
};

```
