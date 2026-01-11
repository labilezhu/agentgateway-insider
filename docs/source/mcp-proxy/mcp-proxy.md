# 代理 MCP 流量

MCP 作为 AI Agent 生态中重要的协议，Agentgateway 对 MCP 的支持是其核心功能之一。本文记录一些在使用 Agentgateway 代理 MCP 流量时的经验和思考。也是我在调查浏览器连不上 Agentgateway MCP 数天后，总结的经验。


[Agentgateway 的官方文档](https://agentgateway.dev/docs/) 有对 MCP 的配置说明，但暂时内容是比较 hello world 级别的简单说明。我的摸索过程，更多是看 agentgateway 源码和看标准规范，包括 3WC 与 MCP 相关的规范。有时间，还需要用 Chrome DevTools debug 一下 javascript 和 network 。



## MCP 基础配置

### STDIO MCP proxy

> https://agentgateway.dev/docs/mcp/connect/stdio/

很多 MCP 工具只支持 STDIO 方式运行，而 Agentgateway 作为一个代理和协议转换器，能将只支持本地使用的 STDIO MCP 工具适配转换为支持远程使用的 Streamable MCP 工具。就算在同一台机器，由于 container 或 vm 隔离的原因，这个把 STDIO MCP 适配为 Streamable MCP 的功能，就是非常实用的。

如，经典的 prometheus MCP 工具 [prometheus-mcp-server](https://github.com/pab1it0/prometheus-mcp-server) ，只能在本地通过 STDIO 方式运行。而通过 Agentgateway 代理后，就能让远程的 AI Agent 通过 Streamable MCP 协议调用 browsermcp 了。

```yaml
config:  

binds:
- port: 3101
  listeners:
  - name: mcp-listener
    hostname: your-host
    routes:
    - name: prometheus
      matches:
      - path:
          pathPrefix: "/mcp/prometheus" #指定 agentgateway 访问地址 path
      policies:
        csrf:
          additionalOrigins:
            - http://your-web-ui-host-origin/
        cors:
          allowOrigins:
            - "*"
          allowHeaders:
            - "*"
            - mcp-protocol-version
            - content-type
            - cache-control
            - Accept
            - mcp-session-id          
          allowCredentials: true
          allowMethods: ["GET", "POST", "OPTIONS"]
          maxAge: 100s
          exposeHeaders:
          - "*"      
          - mcp-session-id          
      backends:
      - mcp:
          targets:
          - name: prometheus
            stdio:
              cmd: docker
              args: ["run","-i","--rm","-e","PROMETHEUS_URL=http://192.168.1.74:9090","ghcr.io/pab1it0/prometheus-mcp-server:latest"]
```

`cors` 与 `csrf` 部分后面会详细说明。这里关注的就是 backend 配置了。不过配置已经很简单，大概不需要展开说明了。

需要说明一下上面 `pathPrefix` 的作用，是在同一个 bind port 监听端口下，以不同的 url path 作为路由原则，路由到不同的 MCP backend server 。其实就是同端口的多路复用。



### Streamable MCP proxy

> https://agentgateway.dev/docs/mcp/connect/http/

```yaml
config:  

binds:
- port: 3101
  listeners:
  - name: mcp-listener
    hostname: your-host
    routes:
    
    - name: home-assistant-route
      matches:
      - path:
          pathPrefix: "/mcp/home-assistant"    
      policies:
        csrf:
          additionalOrigins:
            - http://your-web-ui-host-origin/      
        cors:
          allowOrigins:
            - "*"
          allowHeaders:
            - "*"
            - mcp-protocol-version
            - content-type
            - cache-control
            - Accept
            - mcp-session-id          
          allowCredentials: true
          allowMethods: ["GET", "POST", "OPTIONS"]
          maxAge: 100s
          exposeHeaders:
          - "*"      
          - mcp-session-id          
        requestHeaderModifier:
          add:
            Authorization: "Bearer fake"            
      backends:
      - mcp:
          targets:
          - name: home-assistant
            mcp:
              host: http://192.168.1.68:8123/api/mcp    
```

和 Stdio 类型的 MCP Server 的区别是，多了 `Authorization: "Bearer fake" ` ，这个涉及 MCP 认证，这方面我暂时了解不多。后面有时间再研究了。



## 影响访问的安全配置

有话谚语说：“你不理 politics， 但 politics 会来理你”。数字安全也一样：“就算你不理安全策略，安全策略也有可能妨碍你的访问”。很多软件，默认应用的安全策略均设定了一定的访问门槛，Agentgateway 的 MCP 代理也不例外。我一开始就是踩到这个大坑，花了几天时间才跳出来。

> 注意：由于我不是安全专家，更对前端技术了解有限，以下配置，只作为开发环境使用。生产使用请谨慎。

### CORS 安全配置

> https://agentgateway.dev/docs/configuration/security/cors/

很多 Chat Agent 类型的 Web 应用，考虑到 MCP Server 的网络可达性，以及 OAuth 认证需要用户浏览器参与，会直接在用浏览器访问 MCP Server，而不是后端访问。这时，就是考虑 CORS 访问控制的问题了。其实，包括开发期常用的 [MCP Inspector](https://modelcontextprotocol.io/docs/tools/inspector) 以及 Agentgateway 内置的控制台 A



如果你和我一样是个前端小白，得科普一下什么是 [CORS(Cross-Origin Resource Sharing)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS) 。用后端程序员思维习惯可以理解为：CORS 是一个在浏览器上执行的跨域名(Origin)访问安全策略。策略使用几个规范中定义的 HTTP Header 来陈述。策略的定义在服务端，在浏览器端 javascript 发起 http 访问前作访问限制。

#### 什么是 Origin

> Origin: https://developer.mozilla.org/en-US/docs/Glossary/Origin

Web content's origin is defined by the **scheme** (protocol), **hostname** (domain), and **port** of the URL used to access it. Two  objects have the same origin only when the scheme, hostname, and port all match.  Some operations are restricted to same-origin content, and this restriction can be lifted using CORS.

#### 什么是 CORS(Cross-Origin Resource Sharing)

> CORS: https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS

跨域资源共享 (CORS) 是一种浏览器安全机制，它允许服务器声明哪些 `Orgin` 可以请求资源。CORS 规则是在浏览器端强制执行的，而不是在服务器端。只是在浏览器层拒绝这些请求，如果违反 CORS 策略的 Request 绕过浏览器，直接发送到服务器，服务器是不会检查的。因此，像 curl 这样的工具在处理 CORS 时可能会造成混淆，因为 curl 并不按 CORS  Header 强制执行。



以下是一些相关的 [Preflighted Request Headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS#preflighted_requests):

- Origin - 跨站请求的发起源 Origin
- Access-Control-Request-Method 
- Access-Control-Request-Headers



Reponse Headers:

- [Access-Control-Allow-Origin](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Access-Control-Allow-Origin) - 限制资源的访问权限，仅允许来自指定的 Origin 。
- Access-Control-Allow-Methods - 限制跨站的 http method
- Access-Control-Allow-Headers - 限制跨站 JavaScript 可以定制的 request header
- Access-Control-Max-Age
- Access-Control-Allow-Credentials
- Access-Control-Expose-Headers - 限制跨站 JavaScript 可以读取的 response header



##### Agentgateway 的 CORS 配置

对应于 Agentgateway 的 MCP policies 配置： https://agentgateway.dev/docs/configuration/security/cors/

```yaml
cors:
  allowOrigins:
    - "*"
  allowHeaders:
    - "*"
    - mcp-protocol-version
    - content-type
    - cache-control
    - Accept
    - mcp-session-id          
  allowCredentials: true
  allowMethods: ["GET", "POST", "OPTIONS"]
  maxAge: 100s
  exposeHeaders:
  - "*"      
  - mcp-session-id          
```



### CSRF 配置

#### 什么是 CSRF

> https://developer.mozilla.org/en-US/docs/Web/Security/Attacks/CSRF

上面说的 CSRF 含义广泛，但实际 Agentgateway 用到的，就以下一小块。

##### Agentgateway 的 CSRF 配置

> ##### https://agentgateway.dev/docs/configuration/security/csrf/

```yaml
        csrf:
          additionalOrigins:
            - http://your-web-ui-host-origin/
```

上面 agentgateway 文档写得比较复杂，其实主要是 `additionalOrigins` 要配置上跨站的源 Origin 





