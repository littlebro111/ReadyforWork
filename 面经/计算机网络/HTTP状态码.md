# HTTP 状态码

| 状态码 |             类别              |          原因短语          |
| :----: | :---------------------------: | :------------------------: |
|  1xx   |  Informational 信息性状态码   |     接收的请求正在处理     |
|  2xx   |      Success 成功状态码       |      请求正常处理完毕      |
|  3xx   |   Redirection 重定向状态码    | 需要进行附加操作以完成请求 |
|  4xx   | Client Error 客户端错误状态码 |     服务器无法处理请求     |
|  5xx   | Server Error 服务器错误状态码 |     服务器处理请求出错     |

## 2xx：请求正常处理完毕

* 200 OK：客户端请求成功
* 204 No Content：无内容，服务器处理成功，但是未返回内容
* 206 Partial Content：服务器已经完成了部分 GET 请求（客户端进行了范围请求），响应报文总包含 Content-Range 指定范围的实体内容。

## 3xx：需要进行附加操作以完成请求（重定向）

* 301 Moved Permanently：永久重定向
* 302 Found：临时重定向
* 303 See Other：临时重定向，应使用 GET 定向获取请求资源

* 304 Not Modified：表示客户端发送附带条件的请求，条件不满足时，返回304，不包含响应主体，和重定向没关系。

## 4xx：客户端错误

* 400 Bad Request：客户端请求有语法错误，服务器无法理解
* 401 Unauthorized：请求未经授权，这个状态码必须和 WWW-Authenticate 报头域一起使用
* 403 Forbidden：服务器收到请求，但是拒绝提供服务
* 404 Not Found：请求资源不存在，比如输入了错误的 URL
* 415 Unsupported media type：不支持的媒体类型

## 5xx：服务端错误，服务器未能实现合法的请求

* 500 Internal Server Error：服务器发生不可预期的错误。
* 501 Not Implemented：表示客户端请求的功能还不支持
* 502 Bad Gateway：服务器作为网关或代理时返回的错误码，表示服务器自身工作正常，访问后端服务发生了错误
* 503 Server Unavailable：服务器当前处于超负载或停机维护状态，暂时不能处理客户端请求，一段时间后可能恢复正常