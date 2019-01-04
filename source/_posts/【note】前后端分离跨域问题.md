---
title: 【note】前后端分离跨域问题
date: 2018-10-10 12:39:20
categories: note
tags:
    - React
    - nginx
    - Flask
    - CORS
---

使用React开发可以轻松将前后端分离开来，当前端需要后端提供数据时，可能需要解决跨域问题。这里简单记录下解决跨域问题的一些方法。

* React构建页面，axios请求数据
* Flask提供数据

### 开发环境

使用create-react-app构建的React项目，可以直接在package.json添加代理配置。

```json
 "proxy": "http://localhost:8000/"
```

通过添加代理配置，不需要额外的代码适配。

<!--more-->
### 生产环境

生产环境中使用上述的代理配置无效，通常可以使用以下方式

* CORS
* 反向代理

#### CORS方式

CORS的原理是基于服务方授权的模式，即提供服务的程序要主动通过CORS回应头来声明自己信任哪些源（协议+域名+端口）。得到服务方的授权后，浏览器就可以放行来自这些域的请求了。

* axios 配置

```js
function get(url) {
    let config = {
        url: url,
        method: 'get',
        // 基础url前缀
        baseURL: 'http://api_server:8000/api/',

        // 请求头信息
        headers: {
            'X-Requested-With': 'XMLHttpRequest',
            'Access-Control-Allow-Origin': '*',
            'Accept': 'application/json',
        },

        //设置超时时间
        timeout: 5000,
        //返回数据类型
        responseType: 'json', // default

        // `withCredentials` 表示跨域请求时是否需要使用凭证cookie
        withCredentials: false,
    };
    return axios.get(url, config);
}
```

* Flask 配置

```python
from flask import Flask
from flask import request
from flask import jsonify
from flask_cors import CORS

app = Flask(__name__)
cors = CORS(app, resources={r"/*": {"origins": "*"}}, methods=['GET', 'HEAD', 'POST', 'OPTIONS'])

@app.route("/api/search/")
def search():
    value = request.args.get('key')
    ...
    return jsonify(data)
```

额外安装第三方库flask_cors，配置Flask支持CORS方式

#### 反向代理方式

* nginx配置

  ```
  # nginx配置
  location / {
      index  index.html;
      # React 使用browser history模式
      # url 切换时始终返回index.html
      try_files $uri /index.html;
  }
  
  location ^~ /api/ {
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
      proxy_set_header Host $http_host;
      proxy_redirect off;
      proxy_pass http://127.0.0.1:8000;
  }
  ```

  使用反向代理既可以解决跨域问题，还能隐藏后端api地址。axios配置baseURL为`http://nginx_server/api/`,nginx_server为配置的nginx服务器地址。



利用nginx的rewrite机制，可以方便更改后端请求路径。

```
location /api/ {
            rewrite ^/api/(.*)$ /$1 break;   #所有对后端的请求加一个api前缀方便区分，真正访问的时候移除这个前缀
            # API Server
            proxy_pass http://www.serverA.com;  #将真正的请求代理到serverA,即真实的服务器地址，ajax的url为/api/user/1的请求将会访问http://www.serverA.com/user/1
        }
```


