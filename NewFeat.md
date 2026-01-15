# WARN: 此功能添加可能污染/破坏了原`http.Transport`实现, 使用即代表您已知所有风险.
已修改文件:
- app/cmd/server.go
- app/cmd/server_test.go
- app/cmd/server_test.yaml

# 新功能配置结构示例:
```yaml 
masquerade:
  type: file | proxy | string # 类型: 字符串 (未修改)
  proxy:
    url: http://example.com | https://example.com (也支持基于UDP QUIC的HTTP/3,但官方不保证兼容性) # 类型: 字符串 (未修改)
    rewriteHost: false | true # 类型: 布尔值 (未修改)
    rewriteSNI: false  # 类型: 布尔值 (新增)
    realIpHeader: X-Forwarded-For | X-Real-IP | PROXY Protocol v1 | PROXY Protocol v2 | "Custom Header Name" | None, Disable, False, '' # 类型: 字符串 (新增)
    insecure: false # 类型: 布尔值 (未修改)
```

## masquerade.proxy 新增配置项说明:

### `rewriteSNI`
- **类型**: 布尔值
- **默认值**: `false`
- 说明: |
  是否将发送给masquerade后端的SNI字段重写为`masquerade.proxy.url`的SNI

  **可选值**包括:
  - `false`: 表示连接masquerade时不进行重写, 透传远程客户端请求时提供的SNI字段 (仅 tls / https 协议生效, 无法获取将保持为空)
    设置后`false`程序会劫持 TLS 握手阶段，动态修改 Client Hello 中的 Server Name 字段为客户端提供的 SNI
  - `true`: 则客户端连接时的SNI字段将被替换为`masquerade.proxy.url`的SNI

  设置此项有助于某些需要匹配SNI的代理服务器正确处理请求, 例如 Nginx Stream 模块实现的`Layer 4`不解包 SNI 分流
##### 布尔值正确配置示例?:
  ```yaml
  rewriteHost: true
  ```
  ```yaml
  rewriteHost:True
  ```
  ```yaml
  rewriteHost: TRUE
  ```
  错误写法示例:
  ```yaml
  rewriteHost: "true",
  ```
  ```yaml
  rewriteHost: 'false'
  ```

### `realIpHeader`
- **类型**: 字符串
  ~~**温馨提示**: 字符串类型的内容如果含有特殊符号请使用引号包裹防止解析出错~~
- **默认值**: 无
  将不会传递客户端IP地址
- **说明**: |
  指定用于传递客户端真实IP地址的HTTP头字段名称或协议类型
  **可选值**包括(除自定义头字段名称外不区分大小写):
  - `X-Forwarded`,`X-Forwarded-For`: 常用的HTTP源IP头字段, 用于追加/传递客户端IP地址链. 
    由go`httputil.ReverseProxy`自动处理, 发出时将会自动带上以下字段
    - `X-Forwarded-For`: 客户端IP地址链, 多个IP地址以英文半角逗号`,`分隔.
      示例格式如下:
      ```yaml
      X-Forwarded-For: 192.168.1.1, 192.168.1.100, 172.16.0.1
      ```
    - `X-Forwarded-Proto`: 客户端请求使用的协议 (`http` 或 `https`)
      示例格式如下:
      ```yaml
      X-Forwarded-Proto: https
      ```
    - `X-Forwarded-Host`: 客户端请求时的Host, 与常规HTTP头字段`Host`相同(标准端口号是可省略的)
      示例格式如下:
      ```yaml
      X-Forwarded-Host: example.com:443
      ```
    - `X-Forwarded-Port`: 客户端请求的端口号, (非一定存在, 标准端口号是可省略的)
      示例格式如下:
      ```yaml
      X-Forwarded-Port: 443
      ```
  - `X-Real-IP`: 另一个常见的HTTP源IP头字段, 用于传递客户端IP地址. 与`X-Forwarded`不同的是他仅会保留最后一个客户端IP地址
    示例格式如下:
    ```yaml
    X-Real-IP: 172.16.0.1
    ```
  - `PROXY Protocol`,`PROXY_Protocol`: 保留字段`PROXY Protocol`, 不可用
  - `PROXY Protocol v1`,`PROXY_Protocol_v1`,`PROXY v1`,`PROXY_v1`,`v1`: 使用由HAProxy创造的PROXY协议v1纯文本格式在TCP/UDP中直接传递客户端IP地址
  - `PROXY Protocol v2`,`PROXY_Protocol_v2`,`PROXY v2`,`PROXY_v2`,`v2`: 使用由HAProxy创造的PROXY协议v2二进制格式在TCP/UDP中直接传递客户端IP地址
    *注意*: 使用 PROXY Protocol 时，后端服务器（如 Nginx/Caddy）必须开启对应的PROXY Protocol接收开关，否则连接将会因为无法跳过该头部导致握手失败被连接重置, 进而发生最为常见的连接失败
  - **自定义头字段名称**: 可以指定任意自定义的HTTP头字段名称来传递客户端真实IP地址, 以英文半角连字符`-`作为字段分隔符, 该实现的行为与`X-Real-IP`完全一致
    示例格式如下:
    ```yaml
    realIpHeader: "CF-Connecting-IP"
    ```
    后端将会收到:
    ```yaml
    CF-Connecting-IP: 172.16.0.1
    ```
    温馨提示: 该类型大小写敏感, 写入的字段大小写即为输出的字段名大小写. 
  - 字符串 `None`, `Disable`, `False`, `''`或项不存在: 禁用传递客户端真实IP地址

  配置此项有助于某些需要匹配客户端IP的代理服务器正确处理速率限制

# 其他内容请参阅官方文档: <https://v2.hysteria.network/zh/docs/advanced/Full-Server-Config/#masquerade>

(PS:别问我为什么不PR让大家都用上, 自己看看代码被我改成什么样了...代码风格统一在哪里?性能优化在哪里? 这是能PR的吗? 自己Fork编译一下自己玩就行. 要是真的有需要作者会主动来搬的.)

什么你说你不想编译? 如果你的处理器为amd64(x64)或者aarch64(Arm64), 且操作系统为`MacOS`,`Linux`,`Windows`, 也可以下载个预发行包先玩着.