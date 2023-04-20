# 远程过程调用带来的问题：

1. Call ID 映射
2. 序列化和反序列化   --->   数据编码协议
3. 网络传输   --->   传输协议

牵扯到网络 做成一个 web 服务 （gin、beego、net/httpserver)

这个函数的调用参数如何传递 - json ( json 是一种数据格式的协议)/ xml / protobuf / msgpack，现在网络调用有两个端 - 客户端、应该干嘛？将数据传输到 gin，gin - 服务端，服务端负责解析数据

json 不是一个高性能的编码协议

TCP连接/http协议

http协议 有一个问题：一次性 一旦对方返回了结果 连接断开，http2.0 保持常链接

1. 直接自己基于tcp/udp协议去封装一层协议 myhttp, 没有通用性，http2.0 既有http的特性也有长连接的特性 grpc

http 协议的本质


3-2 替换rpc的序列化协议为json

3-3 替换rpc的传输协议为http

socket

tcp

http

json

RPC 是个概念


