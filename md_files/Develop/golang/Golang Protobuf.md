## Golang Protobuf

在Kafka中，发送的消息是字节数组，因此就需要一个方法来将消息对象序列化为字节数组，在消费者端再反序列化为对象。最常用的序列化格式就是JSON了。虽然JSON对人类非常友好，但是对于机器来说，更容易进行序列化和反序列化的格式还是二进制的格式。

Protobuf（Protocol buffers）是由Google开发的一种二进制协议，用于对结构化数据进行序列化和反序列化。这种格式占用空间更少，更加简单易于维护，同时拥有更好的性能。对于机器之间的通信，Protobuf是比XML和JSON等格式更好的一种选择。

### protoc安装

https://segmentfault.com/a/1190000039767770

### 生成文件

```bash
# -IPATH, --proto_path=PATH   Specify the directory in which to search for
#                              imports.  May be specified multiple times;
#                              directories will be searched in order.  If not
#                              given, the current working directory is used.
#                              If not found in any of the these directories,
#                              the --descriptor_set_in descriptors will be
#                              checked for required proto file.

protoc --go_out=plugins=grpc:. -I=${GOPATH}/src -I=. *.proto

# 生成的目录结构
protoc --go_out=./ test.proto

./proto/test/test.pb.go
```

