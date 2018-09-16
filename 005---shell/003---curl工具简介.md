# curl工具测试RESTful   API  

cURL 可以很方便地完成对 REST API 的调用场景，  
比如：设置 Header，指定 HTTP 请求方法，指定 HTTP 消息体，指定权限认证信息等。  
通过 -v 选项也能输出 REST 请求的所有返回信息。  
cURL 功能很强大，有很多参数，这里列出 REST 测试常用的参数：...

```
-X/--request [GET|POST|PUT|DELETE|…]  指定请求的 HTTP 方法
-H/--header                           指定请求的 HTTP Header
-d/--data                             指定请求的 HTTP 消息体（Body）
-v/--verbose                          输出详细的返回信息
-u/--user                             指定账号、密码
-b/--cookie                           读取 cookie...

```

