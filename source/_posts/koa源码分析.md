---
title: koa源码分析
date: 2017-05-25 15:29:49
tags: koa
categories: nodejs
---
创建一个koa的后端服务只需要3步:
1. 创建koa的app对象
2. 为app添加中间件
3. 监听端口，创建server

下面是一个简单的示例：
```
const Koa = require('koa');
const app = new Koa();

// x-response-time

app.use(async function (ctx, next) {
  console.log('x-response-time start')
  const start = new Date();
  await next();
  const ms = new Date() - start;
  ctx.set('X-Response-Time', `${ms}ms`);
  console.log('x-response-time end')
});

// logger

app.use(async function (ctx, next) {
  console.log('logger start')
  const start = new Date();
  await next();
  const ms = new Date() - start;
  console.log(`${ctx.method} ${ctx.url} - ${ms}`);
  console.log('logger end')
});

// response

app.use(ctx => {
  console.log('hello world')
  ctx.body = 'Hello World';
});

app.listen(3000);

```
当请求http://localhost:3000/时，页面返回'Hello World'
#### 命令行里面顺序打印日志：'x-response-time start' -->  'logger start' --> 'hello world' --> 'logger end' --> 'x-response-time end'
#### 从请求到响应类似下图
![](koa源码分析/koa-onion.png)

创建Koa的app对象，Application继承Emitter对象，

```
 class Application extends Emitter {
  constructor() {
    super();

    this.proxy = false;
    // 用于存储中间件的数组
    this.middleware = [];
    this.subdomainOffset = 2;
    this.env = process.env.NODE_ENV || 'development';
    // 上下文对象
    this.context = Object.create(context);
    // 请求对象
    this.request = Object.create(request);
    // 响应对象
    this.response = Object.create(response);
  }
 }
```
使用app.use()添加中间件
```
  use(fn) {
    //判断fn不是函数返回错误
    if (typeof fn !== 'function') throw new TypeError('middleware must be a function!');
    if (isGeneratorFunction(fn)) {
      deprecate('Support for generators will be removed in v3. ' +
                'See the documentation for examples of how to convert old middleware ' +
                'https://github.com/koajs/koa/blob/master/docs/migration.md');
      fn = convert(fn);
    }
    debug('use %s', fn._name || fn.name || '-');
    //把中间件函数push进application的middleware数组内
    this.middleware.push(fn);
    return this;
  }
```
app.listen()监听端口，listen是createServer()的封装
```
  listen() {
    debug('listen');
    const server = http.createServer(this.callback());
    return server.listen.apply(server, arguments);
  }
```
当服务接收到http请求时，触发callback函数，
```
  callback() {
    const fn = compose(this.middleware);

    if (!this.listeners('error').length) this.on('error', this.onerror);

    const handleRequest = (req, res) => {
      res.statusCode = 404;
      const ctx = this.createContext(req, res);
      const onerror = err => ctx.onerror(err);
      const handleResponse = () => respond(ctx);
      onFinished(res, onerror);
      return fn(ctx).then(handleResponse).catch(onerror);
    };

    return handleRequest;
  }
```
compose用于执行中间件函数，在callback()函数执行fn(ctx)，相当于从dispatch(0)开始，递归执行dispatch(i),直到执行完所有中间件函数
```
function compose (middleware) {
  // 参数判断
  if (!Array.isArray(middleware)) throw new TypeError('Middleware stack must be an array!')
  for (const fn of middleware) {
    if (typeof fn !== 'function') throw new TypeError('Middleware must be composed of functions!')
  }

  return function (context, next) {
    // last called middleware #
    let index = -1
    return dispatch(0)
    function dispatch (i) {
      if (i <= index) return Promise.reject(new Error('next() called multiple times'))
      index = i
      let fn = middleware[i]
      if (i === middleware.length) fn = next
      if (!fn) return Promise.resolve()
      try {
        // 执行中间件函数
        return Promise.resolve(fn(context, function next () {
          return dispatch(i + 1)
        }))
      } catch (err) {
        return Promise.reject(err)
      }
    }
  }
}
```
context用于管理请求，响应
```
  createContext(req, res) {
    const context = Object.create(this.context);
    const request = context.request = Object.create(this.request);
    const response = context.response = Object.create(this.response);
    // 通过context可以获取app,request,response对象
    context.app = request.app = response.app = this;
    context.req = request.req = response.req = req;
    context.res = request.res = response.res = res;
    request.ctx = response.ctx = context;
    request.response = response;
    response.request = request;
    context.originalUrl = request.originalUrl = req.url;
    context.cookies = new Cookies(req, res, {
      keys: this.keys,
      secure: request.secure
    });
    request.ip = request.ips[0] || req.socket.remoteAddress || '';
    context.accept = request.accept = accepts(req);
    context.state = {};
    return context;
  }
```
