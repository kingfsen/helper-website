---
title: "gRPC协议Protocol Buffers"
date: 2019-05-30T16:39:50+08:00
description: "介绍gRPC协议语法以及编码规则，以及其优势"
Tags:
- grpc
Categories:
- golang
---

Protocol Buffers是一种灵活、高效、自动序列化结构数据的协议，当前有两个版本，分别是proto2与proto3，两个版本的协议不能完全兼容。
proto3简化了协议使用，生成的协议使用代码支持更多的编程语言，如Java、C++、Python、Java、Lite、Ruby、JavaScript、Objective-C、C#，推荐使用proto3。

当前已经有应用非常广泛的XML、JSON数据格式，为何Google还会推出Protocol Buffers，并且在Google应用中大量使用，从Protocol Buffers编码可知，与XML、JSON
想必，它具有很多优势。

- 结构简单
- 传输的数据更小
- 处理速度更快
- 生成数据操作方法，在编程中非常容易使用

## 规范

如果使用Protocol Buffers协议，首先需要编写.proto文本文件，通过.proto文件定义消息类型，然后通过编译工具将.proto文件生成不同编程语言的协议使用代码。
.proto文件有一些格式规范建议，其内容也可以当成一种描述语言，在编写时尽量遵循这些规范，这样更容易阅读，编译工具生成的协议编程代码更规范。

- 文件中每行的长度保持在80个字符以内
- 文件名命名成lower_snake_case.proto格式
- 文件中使用2个空格缩进
- 消息名称使用驼峰拼写(首字母大写)，消息中字段使用小写下划线分隔

	```go
	message SongServerRequest {
	  string song_name = 1;
	}
	```

- RPC Service名以及方法名使用驼峰拼写(首字母大写)

	```go
	service FooService {
	  rpc GetSomething(FooRequest) returns (FooResponse);
	}
	```
- 列表字段带上负数形式

	```go
	repeated string keys = 1;
	repeated MyMessage accounts = 17;
	```
	
- Enums枚举类型名采用驼峰拼写(首字母大写)，类型名字采用大写并以下划线分隔，每个类型后面是分号`;`

	```go
	enum Foo {
	  FOO_UNSPECIFIED = 0;
	  FOO_FIRST_VALUE = 1;
	  FOO_SECOND_VALUE = 2;
	}
	```
	
.proto文件与Java中的.java文件一样，内容结构有顺序，proto文件结构:

1. License header (if applicable)
2. File overview
3. Syntax
4. Package
5. Imports (sorted)
6. File options
7. Everything else

## 语法

下面所有的介绍都是基于proto3版本的Protocol Buffers

#### 定义消息类型

```go
syntax = "proto3";

message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
}
```

- syntax指定语法`proto3`，可选的还有`proto2`。
- message定义了一个消息结构`SearchRequest`，消息中定义了三个字段，每个字段都有一个类型、名称以及序号，每个字段均为单一的简单类型。
- 每个字段后面都有一个唯一的数字，这个数字用于在二进制消息流中确定字段值，在消息编码的时候，序号值范围`1`~`15`只需一个字节，首位为标志位，
用于标记一个单元数据后续是否还有数据，后三位用于确定wire类型(最多8种，当前已有5种)，这个wire类型并不是单一的类型，而是某种可以通用转换的类型，
比如int32、int64都是`Varint`类型，后面编码详细介绍。一个字节中只有4位表示字段序号，最大值即为`15`，字段序号范围值`16`~`2047`只需两个字节，
字段序号从`1`开始，最大2<sup>29</sup> - 1，其中`19000`~`19999`被Protocol Buffers保留，不可使用。

上面字段都是单一类型，如果需要定义列表或者数组类型，可以通过`repeated`修饰，在proto3中，`repeated`字段默认使用`packed`编码。在一个proto文件中可以
定义多个消息结构。

```go
message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
}

message SearchResponse {
  repeated string names = 1;
}
```

**注释**
proto文件中注释可以使用`//`或者`/* ... */`语法

```go
/* SearchRequest represents a search query, with pagination options to
 * indicate which results to include in the response. */

message SearchRequest {
  string query = 1;
  int32 page_number = 2;  // Which page number do we want?
  int32 result_per_page = 3;  // Number of results to return per page.
}
```

**保留字段**

在proto文件中可以定义保留字段，字段序号与字段名不能在同一条`reserved`语句中

```go
message Foo {
  reserved 2, 15, 9 to 11;
  reserved "foo", "bar";
}
```

**Package**

在proto文件中可以增加一个可选的package定义，这样在消息类型中就可以避免消息名冲突。

```go
package foo.bar;
message Open { ... }
```

```go
message Foo {
  ...
  foo.bar.Open open = 1;
  ...
}
```

**定义Service**

```go
service SearchService {
  rpc Search (SearchRequest) returns (SearchResponse);
}
```

在proto文件中可以通过import导入另外一个proto文件。


定义了package之后，编译工具在生成对应的使用代码时，包名会有所变化，除非通过option package显示定义文件所在包名。
如果要生成java代码，可以通过`option java_package=com.test`定义包路径，生成go使用代码，可以使用`option go_package`。


#### 字段类型

proto文件中定义的消息字段类型都可以对应到相关编程语言中的字段类型上，下面只展示了Go、Java语言类型对应关系，
更多开发语言中字段对于关系请参阅[官方文档scalar章节](https://developers.google.cn/protocol-buffers/docs/proto3#scalar)。


| proto Type	| Java Type		| Go Type |
| :--------   | :-----  | :----  |
|double		|double			|float64|
|float		|float			|float32|
|int32		|int			|int    |
|int64		|long		    |int64|
|uint32		|int			|uint32|
|uint64		|long			|uint64|
|sint32		|int 			|int32|
|sint64		|long           |int64|
|fixed32	|int			|uint32|
|fixed64	|long			|uint64|
|sfixed32	|int			|	int32|
|sfixed64	|long           | int64|
|bool		|boolean	    |    bool|
|string	    |String			|string|
|bytes		|ByteString		|[]byte|


## 消息编码

要了解Protocol Buffers编码，首先要了解`Base 128 Varints`，`Varints`使用一个或者多个字节去序列化整型。在`varint`中的每个字节(除了最后一个字节)，
每个字节中最高位都是一个标志位，用于标明后续还有更多的字节数据，字节中的后7位称为一个组(group)，然后反转group位置。

现在整型`1`，其二进制为`0000 0001`，由于1只需要一个字节表示，所以标志位默认为0，只有一个group，无须交换位置，最终其varint编码之后即为其二进制。
现在整型`300`，其二进制为`1 0010 1100`，采用varint编码之后为`1010 1100 0000 0010`，编码步骤:

- 以7位分组，`1 0010 1100`分组后为`0000010  0101100`
- 低位组排在前面，交换顺序之后为`0101100 0000010`
- 这是一个完整的数据单元，第一个字节后面还有字节数据，高位添加标志位`1`，第二个字节后面再无数据，高位添加`0`，最终为`10101100 00000010`

现在varint值为`10101100 00000010`，如何得到原始值`300`?

```go
10101100 00000010
#去除标志位
0101100 0000010
#交换group位置
0000010 0101100

100101100

256 + 32 + 8 + 4 = 300
```

这种编码的好处是节省了空间，然后在一些语言中(如Java)，一个int类型数据，需要固定的4个字节，而`varint`却不是固定长度，类似一种可变整型。

#### 消息结构

在Protocol Buffers中，消息是一系列的key-value对，二进制消息中仅仅使用字段序号(Field Number)作为key，字段的名字与类型在解码之后才确定的。
当消息编码之后，keys与values都发送到字节流中进行传输，解码时，解析器必须能够跳过未识别的字段，这样新的字段可以加入消息流中而不影响老版本中的程序运行。
每个key在消息流中都是一个`varint`，其值包含了两个部分`(field_number << 3) | wire_type`，也就是说后三位表示的是`wire type`，各字段类型在消息流中对应的
`wire type`如下。

 ![proto](/blog/go_base/proto.png)
 
**int32字段类型的消息结构**
 
```go
message Test1 {
  int32 a = 1;
}
```
其对应的wire type是0，把a的值设置为150，序列化这个消息到输出流，最终传输的值(十六进制)

```go
08 96 01
```

如果用XML来表示这个消息，与Protocol Buffers相比，XML传输的数据比当前应该要大的多，这就是Protocol Buffers的优势。在Protocol Buffers消息流中，key永远都是一个`varint`，下面分析这个消息数据如何解析成`08 96 01`。

- 字段序号是1，wire type是0，后三位用于表示wire type，key的varint值为`00001000`，即0x08
- 字段a的值为150，根据上面varint编码规则，150编码之后二进制为`1001 0110 0000 0001`，即0x96 0x01
- wire type是0的字段，在消息流中value是紧跟着key，即keyvalue=0x08 0x96 0x01

**string字段类型的消息结构**

```go
message Test2 {
  string b = 2;
}
```
其wire type对应的是2，如果b的值设置为"testing"，消息编码之后值为多少？

- key是varint，后三位表示wire type 2，即010，字段序号也为2，也表示为010，最终key为`0001 0010`，即0x12
- wire type类型为2，key之后跟着的不再是其value，而是value的长度，即"testing"的字节长度7，value长度也是用一个varint`0000 0111`, 即0x07
- 长度之后则是value内容，此时不再是varint，而是字符串中每个字符的acsii码值。

最终消息内容=key + value长度 + value，即十六进制12 07 `74 65 73 74 69 6e 67`，红色即为"testing"的ascii值。

**内嵌消息结构**

```go
message Test3 {
  Test1 c = 3;
}
```

将Test1字段a的值仍然设置为150，编码之后其值？

其wire type也为2，字段序号是3，其key表示为`00011010`，即0x1A，长度为消息Test1的长度，即0x03，编码之后的值为十六进制的1A 03 `08 96 01`。

**数组字段类型的消息结构**

repeated默认的编码采用的是`packed`，其对应的wire type也是2。

```go
message Test4 {
  repeated int32 d = 4;
}
```

给字段`d`提供三个值，分别为3, 270, 86942，编码后的值为

```go
22        // key (field number 4, wire type 2)
06        // payload size (6 bytes)
03        // first element (varint 3)
8E 02     // second element (varint 270)
9E A7 05  // third element (varint 86942)
```

对于整型，采用Varint方式编码非常灵活，相比wire type为1、5的固定长度类型，传输的数据更少。
详细的编码介绍，请参阅[官方文档](https://developers.google.cn/protocol-buffers/docs/encoding)。