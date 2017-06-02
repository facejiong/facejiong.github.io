---
title: 'koa-router源码分析'
date: 2017-05-31 13:55:38
tags: koa-router
categories: nodejs
---
本文koa-router版本是7.2.0

#### 路由是什么？根据请求url路径，通过判断或正则匹配返回对应的页面。
下面是一个简单的原生路由例子：
```
const Koa = require('koa')
const app = new Koa()
const Router = require('koa-router')

var router = new Router()

router.get('/', function (ctx, next) {
  console.log(ctx.router)
  console.log(ctx.params)
  let html = `
    <p>/</p>
  `
  ctx.body = html
})

router.get('/name/:id', function (ctx, next) {
  let html = `
    <p>name:${ctx.params.id}</p>
  `
  ctx.body = html
})

router.get('/company', function (ctx, next) {
  let html = `
    <p>company</p>
  `
  ctx.body = html
})
app.use(router.routes()).use(router.allowedMethods())
console.log(router)
app.listen(3000)
```
#### Router的结构，Router构造函数
```
module.exports = Router;

function Router(opts) {
  if (!(this instanceof Router)) {
    return new Router(opts);
  }

  this.opts = opts || {};
  this.methods = this.opts.methods || [
    'HEAD',
    'OPTIONS',
    'GET',
    'PUT',
    'PATCH',
    'POST',
    'DELETE'
  ];

  this.params = {};
  //stack存储不同的Layer，Router和Layer的关系是Router包含Layer
  this.stack = [];
};
```
Layer构造函数
```
function Layer(path, methods, middleware, opts) {
  this.opts = opts || {};
  this.name = this.opts.name || null;
  this.methods = [];
  this.paramNames = [];
  this.stack = Array.isArray(middleware) ? middleware : [middleware];

  methods.forEach(function(method) {
    var l = this.methods.push(method.toUpperCase());
    if (this.methods[l-1] === 'GET') {
      this.methods.unshift('HEAD');
    }
  }, this);

  // ensure middleware is a function
  this.stack.forEach(function(fn) {
    var type = (typeof fn);
    if (type !== 'function') {
      throw new Error(
        methods.toString() + " `" + (this.opts.name || path) +"`: `middleware` "
        + "must be a function, not `" + type + "`"
      );
    }
  }, this);

  this.path = path;
  this.regexp = pathToRegExp(path, this.paramNames, this.opts);

  debug('defined route %s %s', this.methods, this.opts.prefix + this.path);
};
```
上述例子Router对象
```
Router {
  opts: {},
  methods: [ 'HEAD', 'OPTIONS', 'GET', 'PUT', 'PATCH', 'POST', 'DELETE' ],
  params: {},
  stack:
   [ Layer {
       opts: [Object],
       name: null,
       methods: [Object],
       paramNames: [],
       stack: [Object],
       path: '/',
       regexp: /^(?:\/(?=$))?$/i },
     Layer {
       opts: [Object],
       name: null,
       methods: [Object],
       paramNames: [Object],
       stack: [Object],
       path: '/name/:id',
       regexp: /^\/name\/((?:[^\/]+?))(?:\/(?=$))?$/i },
     Layer {
       opts: [Object],
       name: null,
       methods: [Object],
       paramNames: [],
       stack: [Object],
       path: '/company',
       regexp: /^\/company(?:\/(?=$))?$/i } ] }

```
#### path的匹配分两层，Router遍历所有layer，返回匹配的matched对象
```
Router.prototype.match = function (path, method) {
  var layers = this.stack;
  var layer;
  var matched = {
    //存储path匹配的layer
    path: [],
    //存储methods匹配的layer
    pathAndMethod: [],
    // 是否匹配成功
    route: false
  };

  for (var len = layers.length, i = 0; i < len; i++) {
    layer = layers[i];

    debug('test %s %s', layer.path, layer.regexp);

    if (layer.match(path)) {
      matched.path.push(layer);

      if (layer.methods.length === 0 || ~layer.methods.indexOf(method)) {
        matched.pathAndMethod.push(layer);
        if (layer.methods.length) matched.route = true;
      }
    }
  }

  return matched;
};
```
Layer层通过正则匹配路径
```
Layer.prototype.match = function (path) {
  return this.regexp.test(path);
};
```
#### Router通过use()将methods方法与Router联系起来app.use(router.routes()).use(router.allowedMethods());
router.routes()返回一个中间件，用于对请求发起路由匹配，把一些router参数加入ctx对象,执行router.routes()，返回的是一个dispatch(ctx, next)方法
```
Router.prototype.routes = Router.prototype.middleware = function () {
  var router = this;

  var dispatch = function dispatch(ctx, next) {
    debug('%s %s', ctx.method, ctx.path);
    // 获取path
    var path = router.opts.routerPath || ctx.routerPath || ctx.path;
    // 发起path match，获取matched对象
    var matched = router.match(path, ctx.method);
    var layerChain, layer, i;

    if (ctx.matched) {
      ctx.matched.push.apply(ctx.matched, matched.path);
    } else {
      ctx.matched = matched.path;
    }
    // 可以从ctx 取router
    ctx.router = router;
    // 判断是否匹配成功
    if (!matched.route) return next();

    var matchedLayers = matched.pathAndMethod
    var mostSpecificLayer = matchedLayers[matchedLayers.length - 1]
    ctx._matchedRoute = mostSpecificLayer.path;
    if (mostSpecificLayer.name) {
      ctx._matchedRouteName = mostSpecificLayer.name;
    }

    layerChain = matchedLayers.reduce(function(memo, layer) {
      memo.push(function(ctx, next) {
        ctx.captures = layer.captures(path, ctx.captures);
        ctx.params = layer.params(path, ctx.captures, ctx.params);
        return next();
      });
      return memo.concat(layer.stack);
    }, []);

    return compose(layerChain)(ctx, next);
  };

  dispatch.router = this;

  return dispatch;
};
```
router.allowedMethods()，执行router.allowedMethods()，返回allowedMethods(ctx, next)方法，判断请求的method是否被允许
```
Router.prototype.allowedMethods = function (options) {
  options = options || {};
  var implemented = this.methods;

  return function allowedMethods(ctx, next) {
    return next().then(function() {
      var allowed = {};

      if (!ctx.status || ctx.status === 404) {
        ctx.matched.forEach(function (route) {
          route.methods.forEach(function (method) {
            allowed[method] = method;
          });
        });

        var allowedArr = Object.keys(allowed);
        // 判断请求method是否在允许范围内
        if (!~implemented.indexOf(ctx.method)) {
          if (options.throw) {
            var notImplementedThrowable;
            if (typeof options.notImplemented === 'function') {
              notImplementedThrowable = options.notImplemented(); // set whatever the user returns from their function
            } else {
              notImplementedThrowable = new HttpError.NotImplemented();
            }
            throw notImplementedThrowable;
          } else {
            ctx.status = 501;
            ctx.set('Allow', allowedArr);
          }
        } else if (allowedArr.length) {
          if (ctx.method === 'OPTIONS') {
            ctx.status = 204;
            ctx.set('Allow', allowedArr);
          } else if (!allowed[ctx.method]) {
            if (options.throw) {
              var notAllowedThrowable;
              if (typeof options.methodNotAllowed === 'function') {
                notAllowedThrowable = options.methodNotAllowed(); // set whatever the user returns from their function
              } else {
                notAllowedThrowable = new HttpError.MethodNotAllowed();
              }
              throw notAllowedThrowable;
            } else {
              ctx.status = 405;
              ctx.set('Allow', allowedArr);
            }
          }
        }
      }
    });
  };
};
```
app.use(router.routes()).use(router.allowedMethods());
```
Router.prototype.use = function () {
  var router = this;
  // 将传入参数转换为数组
  var middleware = Array.prototype.slice.call(arguments);
  var path = '(.*)';

  // support array of paths
  if (Array.isArray(middleware[0]) && typeof middleware[0][0] === 'string') {
    middleware[0].forEach(function (p) {
      router.use.apply(router, [p].concat(middleware.slice(1)));
    });

    return this;
  }

  var hasPath = typeof middleware[0] === 'string';
  if (hasPath) {
    path = middleware.shift();
  }

  middleware.forEach(function (m) {
    if (m.router) {
      // 对router.routes()参数的处理
      m.router.stack.forEach(function (nestedLayer) {
        if (path) nestedLayer.setPrefix(path);
        // 绑定layer
        if (router.opts.prefix) nestedLayer.setPrefix(router.opts.prefix);
        router.stack.push(nestedLayer);
      });

      if (router.params) {
        Object.keys(router.params).forEach(function (key) {
          m.router.param(key, router.params[key]);
        });
      }
    } else {
      // 创建并注册一个route
      router.register(path, [], m, { end: false, ignoreCaptures: !hasPath });
    }
  });

  return this;
};
// 创建并注册一个route
Router.prototype.register = function (path, methods, middleware, opts) {
  opts = opts || {};

  var router = this;
  var stack = this.stack;

  // support array of paths
  if (Array.isArray(path)) {
    path.forEach(function (p) {
      router.register.call(router, p, methods, middleware, opts);
    });

    return this;
  }

  // create route Layer
  var route = new Layer(path, methods, middleware, {
    end: opts.end === false ? opts.end : true,
    name: opts.name,
    sensitive: opts.sensitive || this.opts.sensitive || false,
    strict: opts.strict || this.opts.strict || false,
    prefix: opts.prefix || this.opts.prefix || "",
    ignoreCaptures: opts.ignoreCaptures
  });

  if (this.opts.prefix) {
    route.setPrefix(this.opts.prefix);
  }

  // add parameter middleware
  Object.keys(this.params).forEach(function (param) {
    route.param(param, this.params[param]);
  }, this);

  stack.push(route);

  return route;
};
```
