# pwa-learning

> 内部分享service work相关示例以及学习笔记

## service worker

### 1，主要依赖
* `https`: 需要支持https,通过service worker可以劫持连接，伪造和过滤响应，为了避免这些问题，只能在HTTPS的网页上注册service workers，防止加载service worker的时候不被坏人篡改。
* `fetch`: 用于网络请求，Fetch对象暴露在window，替代 XMLHttpRequest,集成promise,或者说基于primise不同于ajax：
  * 1，当接收到一个代表错误的 HTTP 状态码时，从 fetch()返回的 Promise 不会被标记为 reject， 即使该 HTTP 响应的状态码是 404 或 500。相反，它会将 Promise 状态标记为 resolve （但是会将 resolve 的返回值的 ok 属性设置为 false ），仅当网络故障时或请求被阻止时，才会标记为 reject。
  * 2， 默认情况下，fetch 不会从服务端发送或接收任何 cookies, 如果站点依赖于用户 session，则会导致未经认证的请求（要发送 cookies，必须设置 credentials 选项
  * fetch 分为三大模块 Header、Request、Response,可以自行谷歌
    ```

        fetch(url, {
            credentials: 'include'
        })

    ```
* `CacheStorage`: Cache对象存储（本地存储）查看chrome: Application—> cache storage
* `Cache API`: Cache对象暴露在window下,提供Request / Response对象对的存储（增删查改操作）,Cache API 基于promise
    * `Cache.match(request, options)`
        
            返回一个 Promise对象，resolve的结果是跟 Cache 对象匹配的第一个已经缓存的请求。
    * `Cache.matchAll(request, options)`
   
            返回一个Promise 对象，resolve的结果是跟Cache对象匹配的所有请求组成的数组。
    * `Cache.add(request)`
    
            抓取这个URL, 检索并把返回的response对象添加到给定的Cache对象.这在功能上等同于调用 fetch(), 然后使用 Cache.put() 将response添加到cache中.
    * `Cache.addAll(requests)`

            抓取一个URL数组，检索并把返回的response对象添加到给定的Cache对象。

    * `Cache.put(request, response)`
    
            同时抓取一个请求及其响应，并将其添加到给定的cache。

    * `Cache.delete(request, options)` 

            搜索key值为request的Cache 条目。如果找到，则删除该Cache 条目，并且返回一个resolve为true的Promise对象；如果未找到，则返回一个resolve为false的Promise对象。

    * `Cache.keys(request, options)`

            返回一个Promise对象，resolve的结果是Cache对象key值组成的数组。
* 常见的Response属性

    * `Response.status` — 整数(默认值为200) 为response的状态码.

    * `Response.statusText` — 字符串(默认值为"OK"),该值与HTTP状态码消息对应.

    * `Response.ok` — 如上所示, 该属性是来检查response的状态是否在200-299(包括200,299)这个范围内.该属性返回一个Boolean值.

* Request 请求模式

    * `same-origin` — 如果使用此模式向另外一个源发送请求，显而易见，结果会是一个错误。你可以设置该模式以确保请求总是向当前的源发起的。
    * `no-cors` — 保证其对应的方法只有HEAD，GET或POST方法 。即使ServiceWorkers 拦截了这个请求,除了simple header之外不会添加或覆盖任意其他header, 另外JavaScript不会读取Response的任何属性 . 这样将会确保ServiceWorkers不会影响Web语义(semantics of the Web), 同时保证了在跨域时不会发生安全和隐私泄露的问题.

    * `cors` — 允许跨域请求，例如访问第三方供应商提供的各种API。
    
    * `navigate` — 支持导航的一个模式。导航仅供HTML文档使用（在文档之间导航时创建导航请求）。

    _安全起见，只能缓存 GET & HEAD 的请求，对于 POST 等类型请求，返回数据可以保存在 indexDB 中_

* 事件驱动
  * install
  * activate    
  * fetch

## 生命周期
![avatar](/static/sw-life-cycle.png)

## 注册流程

* `register`方法注册service-worker.js(sw存在当前域名根目录下，文件名称自定)，注册后监听service worker提供的生命周期方法—> 
```
!(function (win) {
    const sw = win.navigator.serviceWorker
     
    const killSW = win.killSW || false
 
 
    if (!sw) {
        return
    }
     
    if (!!killSW) {
        sw.getRegistration('/serviceWorker').then(registration => {
            // 手动注销
            registration.unregister()
        })
    } else {
        // 表示该 sw 监听的是根域名下的请求
        sw.register('/serviceWorker.js').then(registration => {
            // 注册成功后会进入回调
            console.log(registration.scope)
        }).catch(err => {
            console.error(err)
        })
    }
})(window)

```

* `install`事件，注册成功后触发，此处主要处理应用内文件的缓存，缓存成功（或者失败）后返回promise，注册事件结束—> 

```
var CACHE_NAME = 'my-site-cache-v1';
var urlsToCache = [
  '/',
  '/styles/main.css',
  '/script/main.js'
];

self.addEventListener('install', function(event) {
  // Perform install steps
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(function(cache) {
        console.log('Opened cache');
        return cache.addAll(urlsToCache);
      })
  );
});
// self 是内置全局变量，可以理解为代码service worker服务本身，类似于window

// event.waitUntil 接受一个promise，通常是缓存是否成功，安装是否完成,promise reject，则表明安装失败

```

* `activate`事件，当前已无其他处于激活状态的Service Worker - 通过调用self.skipWaiting()强制激活 - 此前处于激活状态的Service Worker已经过期(通过刷新页面可以使旧的Service Worker过期)触发，此处主要用于做版本控制，一旦service-worker文件变化，service worker激活时检查过期缓存并删除（quesition: 为什么不在install的时候就检查并删除过期缓存，tips：1 预缓存和脏检查分离，2注册未完成无法检索哪些是新缓存，哪些是过期缓存 3注册很可能会失败）—> 

```

self.addEventListener('activate', function(event) {

  var cacheWhitelist = ['pages-cache-v1', 'blog-posts-cache-v1'];

  event.waitUntil(
    caches.keys().then(function(cacheNames) {
      return Promise.all(
        cacheNames.map(function(cacheName) {
          if (cacheWhitelist.indexOf(cacheName) === -1) {
            // 浏览器缓存 caches 会一直保存存存存到存不动了，再去删除某些资源，这个是浏览器的行为，因此还是建议在每次更改后去删除一些旧的浏览器资源，可以自己设定
            return caches.delete(cacheName);
          }
        })
      );
    })
  );
});

```

* `fetch`事件，激活成功后service运行在浏览器后台，并开始事实上接管页面的所有网络请求(fetch)，此处主要是监测网络请求在缓存中比对，如果已经被缓存直接返回，否则正常,另：此处可以用来做白名单控制—>

```

self.addEventListener('fetch', event => {
    let { request } = event
 // event.respondWith 接收的是一个 promise 参数，把其结果返回到 client 中
    event.respondWith(
        // 先从 caches 中寻找是否有匹配
        caches.match(request).then(res => {
            if (res) {
                return res
            }
     
            // 对于 CDN 资源要更改 request 的 mode
            if (request.mode !== 'navigate' && request.url.indexOf(request.referrer) === -1) {
                request = new Request(request, { mode: 'cors' })
            }
             
            // 对于不在 caches 中的资源进行请求
            return fetch(request).then(fetchRes => {
                // 这里只缓存成功 && 请求是 GET 方式的结果，对于 POST 等请求，可把 indexDB 给用上
                if(!fetchRes || fetchRes.status !== 200 || request.method !== 'GET') {
                    return fetchRes
                }
 // Request & Response 中的 body 只能被读取一次，究其原因，是其中包含 bodyUsed 属性，当使用过后，这个属性值就会变为 true， 不能再次读取，解决方法是，把 Request & Response clone 下来：request.clone() || response.clone()
                let resClone = fetchRes.clone()
 
                caches.open(CACHE_NAME).then(cache => {
                    cache.put(request, fetchRes)
                })
 
                return resClone
            })
        })
    )
})

// event.respondWith
// 接受一个promise,通常是一个请求结果（可以是缓存读取或者正常网络请求，由开发者控制）,拿到结果后响应给客户端


```

* `Redundant`事件，installing 事件失败 - activating 事件失败 - 被新的Service Worker取代都会触发废弃事件

## 示例代码
[点击查看](/dist/service-worker.js)

## 调试方法
[点击查看](https://lavas.baidu.com/pwa/offline-and-cache-loading/service-worker/service-worker-debug)