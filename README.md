# pwa-learning

> 内部分享service work相关示例以及学习笔记

## service worker

### 1，主要依赖
* `https`: 需要支持https,通过service worker可以劫持连接，伪造和过滤响应，为了避免这些问题，只能在HTTPS的网页上注册service workers，防止加载service worker的时候不被坏人篡改。
* `fetch`: 用于网络请求，Fetch对象暴露在window，替代 XMLHttpRequest,集成promise,或者说基于primise不同于ajax：
  * 1，当接收到一个代表错误的 HTTP 状态码时，从 fetch()返回的 Promise 不会被标记为 reject， 即使该 HTTP 响应的状态码是 404 或 500。相反，它会将 Promise 状态标记为 resolve （但是会将 resolve 的返回值的 ok 属性设置为 false ），仅当网络故障时或请求被阻止时，才会标记为 reject。
  * 2， 默认情况下，fetch 不会从服务端发送或接收任何 cookies, 如果站点依赖于用户 session，则会导致未经认证的请求（要发送 cookies，必须设置 credentials 选项。
* `CacheStorage`: Cache对象存储（本地存储）查看chrome: Application—> cache storage
* `Cache API`: Cache对象暴露在window下,提供Request / Response对象对的存储（增删查改操作）
    * caches.open
    * cache.addAll 抓取一个url数组，检索并将
    * cache.put 添加缓存
    * caches.match 缓存匹配
    * caches.delete 删除过期的缓存
    * caches.keys 

    _对URL寻址资源，使用Cache API。对其他数据，使用IndexedDB。_

## 生命周期
![avatar](/static/sw-life-cycle.png)

## 注册流程

* `register`方法注册service-worker.js(sw存在当前域名根目录下，文件名称自定)，注册后监听service worker提供的生命周期方法—> 

* `install`事件，注册成功后触发，此处主要处理应用内文件的缓存，缓存成功（或者失败）后返回promise，注册事件结束—> 

* `actiavte`事件，当前已无其他处于激活状态的Service Worker - 通过调用self.skipWaiting()强制激活 - 此前处于激活状态的Service Worker已经过期(通过刷新页面可以使旧的Service Worker过期)触发，此处主要用于做版本控制，一旦service-worker文件变化，service worker激活时检查过期缓存并删除（quesition: 为什么不在install的时候就检查并删除过期缓存，tips：1 预缓存和脏检查分离，2注册未完成无法检索哪些是新缓存，哪些是旧的缓存 3注册很可能会失败）—> 

* `fetch`事件，激活成功后service运行在浏览器后台，并开始事实上接管页面的所有网络请求(fetch)，此处主要是监测网络请求在缓存中比对，如果已经被缓存直接返回，否则正常,另：此处可以用来做白名单控制—>

* `Redundant`事件，installing 事件失败 - activating 事件失败 - 被新的Service Worker取代都会触发废弃事件

## 事件提供的回调callback
 
 * event.waitUntil 接受一个promise，通常是缓存是否成功，安装是否完成,promise reject，则表明安装失败

* event.respondWith
接受一个promise,通常是一个请求结果（可以是缓存读取或者正常网络请求，由开发者控制）