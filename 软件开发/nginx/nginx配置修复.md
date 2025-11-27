## nginx配置修复实例

### 1. 修复foo路径返回404问题
原配置：
```
location /foo {
    root /data/foo/dist;
}
```

修改后配置：
```
location /foo/ {
    alias /data/foo/dist/;
    try_files $uri $uri/ /index.html;
}
```

重点：
- /foo/ 结尾必须带 /
- alias 路径也要 /
- Vue/React 必须加 try_files 才能正常访问

访问方式：
http://localhost:8888/foo/

原理解析：
1. 原配置中root 的拼接规则是：

文件实际路径 = root路径 + location匹配到的完整路径

也就是 Nginx 会尝试访问：

/data/foo/dist/foo/index.html，但实际文件路径是：/data/foo/dist/index.html，因此路径错误，返回404。

2. 修改后的配置中，alias 路径是：/data/foo/dist/，Nginx 会尝试访问：

/data/foo/dist/index.html，路径正确，返回文件内容。

3. alias的作用是：
- 直接将请求路径映射到 alias 路径下的文件或目录，而不包含 location 匹配到的路径部分。
- 因此，在 Vue/React 等单页应用中，需要使用 alias 来指定静态资源的路径，否则会返回 404 错误。

4. 注意事项
- alias 路径必须以 / 结尾，否则会导致路径错误。
- try_files 指令必须包含 /index.html，否则会导致 Vue/React 等单页应用无法正常访问。

### 2. 请求后端接口报401错误
#### 2.1 问题描述
http://localhost:8888/prod-api/captchaImage 接口报401，{"msg":"请求访问：/prod-api/captchaImage，认证失败，无法访问系统资源","code":401}。后端的白名单是/captchaImage，也就是反向代理到后端时URL多出了/prod-api

#### 2.2 问题原因
proxy_pass 没有尾斜杠 → Nginx 会把 /prod-api 保留下来传给后端
于是浏览器访问：

/prod-api/captchaImage


后端实际收到的是：

/prod-api/captchaImage


而它希望收到的是：

/captchaImage

#### 2.3 修复方案

你需要：

让后端收到 去掉 /prod-api 前缀后的路径

正确配置：
```
location /prod-api/ {
    proxy_pass http://localhost:8880/;

    proxy_set_header Host $http_host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header REMOTE-HOST $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    proxy_http_version 1.1;
    proxy_set_header Connection '';
    chunked_transfer_encoding on;

    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;

    proxy_buffering off;
    proxy_cache off;
}
```


注意这两个细节 非常关键：

location /prod-api/ # 末尾一定要有 /

proxy_pass http://localhost:8880/;  # 这里也必须带 /

### 3. 接口可以访问，但是登录成功后直接跳转到404页面

### 3.1 问题描述
接口口可以访问了，但是登录成功后直接跳转到404页面，正常应该跳转到index。怀疑还是foo前缀的问题，前端地址现在还是/index而不是/foo/index

### 3.2 问题原因
前端路由配置问题，默认是/index，而后端返回的是/foo/index。

### 3.3 修复方案
vue.config.js 添加：

```js
module.exports = {
  publicPath: '/foo/'
}
```

修改 src/router/index.js

找到：
```js
const router = new Router({
  mode: 'history',
  scrollBehavior: () => ({ y: 0 }),
  routes: constantRoutes
})
```

改成：

```js
const router = new Router({
  mode: 'history',
  base: process.env.NODE_ENV === 'production' ? '/foo/' : '/',
  scrollBehavior: () => ({ y: 0 }),
  routes: constantRoutes
})
```
注意别忘了重新build项目。

### 4. Vue单页面应用，第一次访问正确，但是在页面刷新后会提示404

#### 4.1 问题描述
Vue单页面应用，第一次访问正确，但是但是URL如http://localhost:8888/foo/project/base/hub在页面刷新后会提示404。

#### 4.2 问题原因
这个现象100%是因为 Vue-Router history 模式 + Nginx 没有做 history 回退处理。
这是典型问题：

第一次访问 OK，刷新 404

因为刷新时浏览器请求：

/foo/project/base/hub


Nginx 会在服务器上去找这个路径对应的真实文件夹
但实际文件只有 /data/foo/dist/index.html

所以 → 404

#### 4.3 修复方案
给 /foo/ 做 SPA fallback

修改你的 Nginx：

```
location /foo/ {
    alias /data/foo/dist/;
    try_files $uri $uri/ /foo/index.html;
}
```


⚠ 注意：
try_files 的最终 fallback 必须是 /foo/index.html

而不是 /index.html

也不是 /data/foo/dist/index.html

#### 4.4 原理解析
第一次访问

你在浏览器输入：

http://ip/foo/project/base/hub


步骤：

1. 浏览器访问 server index.html
2. index.html加载 Vue
3. Vue Router 看到 URL
4. SPA 渲染对应的页面

这个时候，服务器不关心 /project/base/hub，只有前端在用它

但刷新发生了什么？

刷新时，浏览器会重新请求：

GET /foo/project/base/hub HTTP


服务器想：

嗯？我来硬盘上找这个目录看看？

自然找不到

所以直接 404

因为服务器根本不知道这是 SPA 的虚拟路径

🌈 try_files 就是：

如果你查不到这个目录，就去返回主页

让前端接管路由

📌 用一句话解释 try_files
```
try_files $uri $uri/ /foo/index.html;
```


意思就是：

> 试试有没有这个文件
> 再试试是不是个文件夹
> 都没有，那就给 index.html