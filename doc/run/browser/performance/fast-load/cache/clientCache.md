# Cache
我们知道HTTP的缓存属于客户端缓存，后面会提到为什么属于客户端缓存。所以我们认为浏览器存在一个缓存数据库，用于储存一些不经常变化的静态文件（图片、css、js等）。我们将缓存分为强制缓存和协商缓存。下面我将分别详细的介绍这两种缓存的缓存规则。

## 缓存策略
### 强制缓存
当缓存数据库中已有所请求的数据时。客户端直接从缓存数据库中获取数据。当缓存数据库中没有所请求的数据时，客户端的才会从服务端获取数据。触发强制缓存后，Firefox浏览器表现为一个灰色的200状态码。chrome为200 (from disk cache)或是200 OK (from memory cache)，这里chrome会根据你的内存情况来选择存在内存还是硬盘
+ Expires
+ Cache-Control
    + max-age=[秒]	缓存的时长，也是响应的最大的Age值
    + no-store	不缓存请求或是响应的任何内容
    + no-cache	强制源服务器再次验证缓存是否可用
    + max-age=0和no-cache应该是从语气上不同。max-age=0是告诉客户端资源的缓存到期应该向服务器验证缓存的有效性。而no-cache则告诉客户端使用缓存前必须向服务器验证缓存的有效性。
    + private 客户端可以缓存 
    + public 客户端和代理服务器都可以缓存 
+ Cache-Control优先级大于Expires

### 协商缓存
又称对比缓存，客户端会先从缓存数据库中获取到一个缓存数据的标识，得到标识后请求服务端验证是否失效（新鲜），如果没有失效服务端会返回304，此时客户端直接从缓存中获取所请求的数据，如果标识失效，服务端会返回更新后的数据。

两类缓存机制可以同时存在，强制缓存的优先级高于协商缓存，当执行强制缓存时，如若缓存命中，则直接使用缓存数据库数据，不在进行缓存协商。



+ Last-Modified
    + If-Moified-Since
    + 在浏览器第一次请求某一个URL时，服务器端的返回状态码会是200，响应的实体内容是客户端请求的资源，同时有一个Last-Modified的属性标记此文件在服务器端最后被修改的时间。`Last-Modified : Fri , 12 May 2006 18:53:33 GMT`
    + 当浏览器第二次请求这个URL的时候，根据HTTP协议规定，浏览器会把第一次Last-Modified的值存储在If-Modified-Since里面发送给服务端来验证资源有没有修改。`If-Modified-Since : Fri , 12 May 2006 18:53:33 GMT`
    + 服务端通过If-Modified-Since字段来判断在这两次访问期间资源有没有被修改过，从而决定是否返回完整的资源。如果有修改正常返回资源，状态码200，如果没有修改只返回响应头，状态码`304`，告知浏览器资源的本地缓存还可用。
    + Last-Modified有几个缺点：没法准确的判断资源是否真的修改了，比如某个文件在1秒内频繁更改了多次，根据Last-Modified的时间(单位是秒)是判断不出来的，再比如，某个资源只是修改了，但实际内容并没有发生变化，Last-Modified也无法判断出来，因此在HTTP/1.1中还推出了ETag这个字段👇

+ Etag
    + If-None-Match
    + 在浏览器第一次请求某一个URL时，服务器可以通过某种自定的算法对资源生成一个唯一的标识(比如md5标识)，
    并返给浏览器`ETag: abc-123456`
    + 当浏览器第二次请求这个URL的时候，根据HTTP协议规定，浏览器回把第一次ETag的值存储在If-None-Match里面发送给服务端来验证资源有没有修改。`If-None-Match: abc-123456`
    + Get请求中，当且仅当服务器上没有任何资源的ETag属性值与这个首部中列出的相匹配的时候，服务器端会才返回所请求的资源，响应码为200。如果没有资源的ETag值相匹配，那么返回`304`状态码。
    + `POST、PUT`等请求改变文件的请求，如果没有资源的ETag值相匹配，那么返回`412`状态码。
    + 当然和Last-Modified相比，ETag也有自己的缺点，比如由于需要对资源进行生成标识，性能方面就势必有所牺牲。
+ If-None-Match优先级大于Last-Modified
### 启发式缓存
```
Age:23146
Cache-Control: public
Date:Tue, 28 Nov 2017 12:26:41 GMT
Last-Modified:Tue, 28 Nov 2017 05:14:02 GMT
Vary:Accept-Encoding
```

如果浏览器用来确定缓存过期时间的字段一个都没有！那该怎么办？启发式缓存！
根据响应头中2个时间字段 Date 和 Last-Modified 之间的时间差值，取其值的10%作为缓存时间周期。



## 用户行为
+ 打开新窗口
如果指定cache-control的值为private、no-cache、must-revalidate,那么打开新窗口访问时都会重新访问服务器。而如果指定了max-age值,那么在此值内的时间里就不会重新访问服务器
+ 在地址栏回车
如果值为private或must-revalidate,则只有第一次访问时会访问服务器,以后就不再访问。如果值为no-cache,那么每次都会访问。如果值为max-age,则在过期之前不会重复访问。
+ 按后退按扭
如果值为private、must-revalidate、max-age,则不会重访问,而如果为no-cache,则每次都重复访问.
+ 按刷新按扭 F5
无论为何值,都会重复访问.请求带上If-Modify-since，去服务器看看这个文件是否有过期了
+ 按强制刷新按钮 CTRL/F5
浏览器删除缓存，当做首次进入重新请求(返回状态码200)

## 实践
> 最佳实践，就应该是尽可能命中强缓存，同时，能在更新版本的时候让客户端的缓存失效。

### 前端打包加hash
webpack给我们提供了三种哈希值计算方式，分别是hash、chunkhash和contenthash
+ hash
    + 只要有一个文件变化，整个项目都hash都会变，显然不行
    ```js
        const path = require('path');
        module.exports = {
            entry: {
                index: './src/index.js',
                main: './src/main.js'
            },
            output: {
                path: path.resolve(__dirname, 'dist'),
                filename: '[name].[hash].js',
            }
        }
    ```
+ chunkhash
    + 同一模块中一个文件变化，整个模块hash变化。看起来还行，不过如果js和css是分开了的，只要其中一个变化，整个模块的js和css都会变
    ```js
        module.exports = {
            output: {
                path: path.resolve(__dirname, 'dist'),
                filename: '[name].[chunkhash].js',
            }
        }
    ```
+ contenthash
    + 只要文件内容不一样，文件产生的哈希值就不一样，显然这就是我们想要的
    ```js
        module.exports = {
            output: {
                path: path.resolve(__dirname, 'dist')
                filename: '[name].[contenthash].js',
            }
        };
    ```
### Nginx
+ 官方默认开启ETag，所以这里就不用做特别设置了

### 后端
+ 强制缓存
```js
res.setHeader('Cache-Control', 'public, max-age=xxx');
```
+ 协商缓存
```js
res.setHeader('Cache-Control', 'public, max-age=0');
res.setHeader('Last-Modified', xxx);
res.setHeader('ETag', xxx);
```
+ 当然也有很多库,比如koa的koa-static
```js
const KoaStatic = require('koa-static');
app.use(KoaStatic('./',{ maxAge: 365 * 24 * 60 * 60 }));
```