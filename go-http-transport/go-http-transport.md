## Go HTTP Transport 源码剖析

### 前言
`Transport`是Go HTTP客户端的核心，Go HTTP Client 可以分为两层：
- 第一层是参数层，处理请求的参数，包括参数构造，校验，格式化等，由`http.Request`实现。
- 第二层是网络层，职责是建立连接，发送请求，维护连接，读取返回值等，由`http.Transport`实现。
