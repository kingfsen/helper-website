---
title: "使用Golang部署gRPC服务"
date: 2019-05-31T16:51:12+08:00
description: "grpc的基础概念以及服务构建详细介绍"
thumbnail: "img/grpc.png"
Tags:
- grpc
Categories:
- golang
---

gRPC与大多数RPC框架一样，通过定义一个服务Service，然后明确指定能够被远程调用的方法。gRPC默认使用Protocol Buffers作为接口定义语言以及消息传输格式，
当然也可以使用其它可替代的协议，关于Protocol Buffers的介绍请参考[gRPC协议Protocol Buffers](/post/protocol_buffers/)。

gRPC可以定义四种类型的服务方法

- **Unary RPCs** 客户端发送一个请求到服务端，然后从服务端得到一个返回的响应，类似一个普通方法的调用。

```go
rpc SayHello(HelloRequest) returns (HelloResponse){
}
```

- **Server streaming RPCs** 客户端发送一个请求到服务端，然后得到一个流去读取返回的一系列消息。客户端从返回的流中读取数据直到流中没有数据可读取，在单个
独立的调用过程中gRPC保证了消息的时序性。

```go
rpc LotsOfReplies(HelloRequest) returns (stream HelloResponse){
}
```

- **Client streaming RPCs** 客户端通过提供的一个流，写一系列的消息并发送到服务端。客户端一旦完成了消息的写入，则等待服务端读取消息获取返回结果，在单个
独立的调用过程中gRPC保证了消息的时序性。

```go
rpc LotsOfGreetings(stream HelloRequest) returns (HelloResponse) {
}
```

- **Bidirectional streaming RPCs** 客户端与服务端分别使用一个读写流发送一系列的消息，两个流操作独立，读写顺序无任何要求，如服务端在写消息之前，可以等待接收
客户端发送的所有消息，活着读一个消息，然后发送一个消息。

```go
rpc BidiHello(stream HelloRequest) returns (stream HelloResponse){
}
```

通过编译工具生成代码之后，服务端需要实现Service的相关方法，客户端通过一个存根对象直接调用Service的方法。

## 编写proto文件

首先编写一个student.proto文件 

```go
syntax = "proto3";

option go_package = "sunjinfu/api/proto";

import "google/protobuf/empty.proto";


message SearchRequest {
  string id = 1;
  string name = 2;
}

message SearchResponse {
  repeated Student students = 1;
}

message Student {
  int64 id = 1;
  string name = 2;
  Address address = 3;
}

message Address {
  string street = 1;
  string postcode = 2;
}

service StudentService {

  rpc GetAllStudents(google.protobuf.Empty) returns (SearchResponse);
  
  rpc AddStudent(Student) returns (google.protobuf.Empty);
  
  rpc SearchStudent(SearchRequest) returns (Student);
  
  rpc GetStudent(stream SearchRequest) returns (SearchResponse);
}
```


## 生成代码

- 安装Protocol Buffers v3编译工具

编译工具下载地址: https://github.com/protocolbuffers/protobuf/releases ，请选择对应的系统版本下载，比如windows选择`protoc-3.8.0-win64`，然后解压文件，
bin目录下有可执行程序protoc，将其路径添加到环境变量PATH中。include目录下是google自带的一些proto文件，这些proto文件定义了一些公共的消息类型，
例如上面student.proto导入了google/protobuf/empty.proto，在编译的时候必须把这个目录带上，否则会编译报错，找不到文件。

- 安装编译插件protoc-gen-go

```go
go get -u github.com/golang/protobuf/protoc-gen-go
```
protoc-gen-go安装成功之后，可执行程序文件路径为$GOPATH/bin，请把$GOPATH/bin添加到系统环境变量PATH中，否则protoc在编译proto文件时无法找到插件。

以本机windows为例，把student.proto移动到E:\grpc\proto下，同时把protoc官方proto包(google/protobuf)移动到E:\grpc\include目录下。

```bash
$ protoc -IE:/grpc/proto -IE:/grpc/include --go_out=plugins=grpc:. student.proto
```

命令执行成功之后，则在当前命令执行的路径下生成定义的包sunjinfu/api/proto，包下对应的文件student.pb.go，一般不要手动去修改该文件中的内容。

> * 注意:如果--go_out未添加plugins，则pb.go文件中不会生成相关Service相关服务的调用方法，只有消息类型相关处理方法。

## 实现服务端代码

编写grpc服务代码时依赖了google的grpc包，需要科学上网获取该包，以及该包的一些依赖包，主要是golang.org/x等。

```go
go get -u google.golang.org/grpc
```

新建一个golang项目，将上述生成的包sunjinfu/api/proto添加到$GOPATH中，从生成student.pb.go文件中可知，服务端只需实现如下四个接口即可。

```go
// StudentServiceServer is the server API for StudentService service.
type StudentServiceServer interface {
	GetAllStudents(context.Context, *empty.Empty) (*SearchResponse, error)
	AddStudent(context.Context, *Student) (*empty.Empty, error)
	SearchStudent(context.Context, *SearchRequest) (*Student, error)
	GetStudent(StudentService_GetStudentServer) error
}
```

编写一个main.go文件，实现以上四个接口，这里简单Mock返回数据，然后启动gRPC服务端。

```go
package main

import (
	"context"
	"github.com/golang/protobuf/ptypes/empty"
	"google.golang.org/grpc"
	"io"
	"log"
	"net"
	"sunjinfu/api/proto"
)

const (
	port = ":50051"
)

type server struct {}

func (s *server) GetAllStudents(ctx context.Context, req *empty.Empty) (*proto.SearchResponse, error) {
	log.Printf("GetAllStudents invoke")
	//Mock business to get data
	var students []*proto.Student
	s1 := &proto.Student{
		Id: 1,
		Name: "开发者",
		Address: &proto.Address{
			Street: "Zh.Load",
			Postcode: "100020",
		},
	}
	s2 := &proto.Student{
		Id: 2,
		Name: "测试者",
		Address: &proto.Address{
			Street: "En.Load",
			Postcode: "100021",
		},
	}
	students = append(students, s1, s2)

	response := &proto.SearchResponse{Students: students}

	return response, nil
}

func (s *server) AddStudent(ctx context.Context, req *proto.Student) (*empty.Empty, error) {
	log.Printf("AddStudent invoke, param: %+v", *req)
	//Mock business logic
	log.Printf("Add student successfully")
	return nil, nil
}

func (s *server) SearchStudent(ctx context.Context, req *proto.SearchRequest) (*proto.Student, error) {
	log.Printf("SearchStudent invoke, param: %+v", *req)

	//Mock business search logic
	log.Printf("Search student successfully")

	return &proto.Student{
		Id: 2,
		Name: "测试者",
		Address: &proto.Address{
			Street: "En.Load",
			Postcode: "100021",
		},
	}, nil

}

func (s *server) GetStudent(stream proto.StudentService_GetStudentServer) error {
	log.Printf("GetStudent invoke")
	for {
		sr, err := stream.Recv()
		if err == io.EOF {
			//Mock return data
			var students []*proto.Student
			s1 := &proto.Student{
				Id: 1,
				Name: "开发者",
				Address: &proto.Address{
					Street: "Zh.Load",
					Postcode: "100020",
				},
			}
			s2 := &proto.Student{
				Id: 2,
				Name: "测试者",
				Address: &proto.Address{
					Street: "En.Load",
					Postcode: "100021",
				},
			}
			students = append(students, s1, s2)
			return stream.SendAndClose(&proto.SearchResponse{
				Students: []*proto.Student{s1, s2},
			})
		}
		log.Printf("Receive message: %v", *sr)
		if err != nil {
			log.Printf("Receive message occured error: %v", err)
			return err
		}
	}
}

func main() {
	lis, err := net.Listen("tcp", port)
	if err != nil {
		log.Fatalf("Failed to listen: %v", err)
	}
	s := grpc.NewServer()
	proto.RegisterStudentServiceServer(s, &server{})
	if err := s.Serve(lis); err != nil {
		log.Fatalf("Failed to serve: %v", err)
	}
}
```
运行main.go文件，成功启动之后，gRPC则在端口50051提供服务。

## 实现客户端代码

编写一个main文件，调用gRPC服务端，注意使用stream请求参数类型的方法调用。

```go
package main

import (
	"context"
	"github.com/golang/protobuf/ptypes/empty"
	"google.golang.org/grpc"
	"log"
	"strconv"
	"strings"
	"sunjinfu/api/proto"
	"time"
)

const (
	address = "localhost:50051"
)

func main() {
	conn, err := grpc.Dial(address, grpc.WithInsecure())
	if err != nil {
		log.Printf("Can not connect gRPC server: %v", err)
	}
	defer conn.Close()
	c := proto.NewStudentServiceClient(conn)
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()
	response, err := c.GetAllStudents(ctx, &empty.Empty{})
	if err != nil {
		log.Fatalf("Failed to invoke GetAllStudents, err: %v", err)
	}
	log.Printf("GetAllStudents response message: %v", *response)

	student, err := c.SearchStudent(ctx, &proto.SearchRequest{Name: "test"})
	if err != nil {
		log.Printf("Failed to invoke SearchStudent, err: %v", err)
	}
	log.Printf("SearchStudent response message: %v", *student)

	_, err = c.AddStudent(ctx, &proto.Student{
			Id: 10,
			Name: "sunjinfu",
			Address: &proto.Address{
				Street: "Load",
				Postcode: "100020",
			},
		})
	if err != nil {
		log.Printf("Failed to invoke AddStudent, err: %v", err)
	}

	//注意这个请求参数是stream类型的方法
	stream, err := c.GetStudent(ctx)
	if err != nil {
		log.Printf("Failed to invoke  GetStudent, err: %v", err)
	} else {
		//发送10个SearchRequest消息
		for a := 0; a < 10; a++ {
			sr := &proto.SearchRequest{
				Id: strconv.Itoa(a),
				Name: strings.Join([]string{"name", strconv.Itoa(a)}, "")}
			err = stream.Send(sr)
			if err != nil {
				log.Printf("Failed to send message, err: %v", err)
			}
		}
		reply, err := stream.CloseAndRecv()
		if err != nil {
			log.Printf("Failed to get reply message from server, err: %v", reply)
		} else {
			log.Printf("GetStudent reply message: %v", *reply)
		}

	}
}
```

查看客户端调用日志

```bash
2019/06/01 08:53:01 GetAllStudents response message: {[id:1 name:"\345\274\200\345\217\221\350\200\205" address:<street:"Zh.Load" postcode:"100020" >  id:2 name:"\346\265\213\350\257\225\350\200\205" address:<street:"En.Load" postcode:"100021" > ] {} [] 0}
2019/06/01 08:53:01 SearchStudent response message: {2 测试者 street:"En.Load" postcode:"100021"  {} [] 0}
2019/06/01 08:53:01 GetStudent reply message: {[id:1 name:"\345\274\200\345\217\221\350\200\205" address:<street:"Zh.Load" postcode:"100020" >  id:2 name:"\346\265\213\350\257\225\350\200\205" address:<street:"En.Load" postcode:"100021" > ] {} [] 0}
```

查看gRPC服务端日志

```bash
2019/06/01 08:53:01 GetAllStudents invoke
2019/06/01 08:53:01 SearchStudent invoke, param: {Id: Name:test XXX_NoUnkeyedLiteral:{} XXX_unrecognized:[] XXX_sizecache:0}
2019/06/01 08:53:01 Search student successfully
2019/06/01 08:53:01 AddStudent invoke, param: {Id:10 Name:sunjinfu Address:street:"Load" postcode:"100020"  XXX_NoUnkeyedLiteral:{} XXX_unrecognized:[] XXX_sizecache:0}
2019/06/01 08:53:01 Add student successfully
2019/06/01 08:53:01 GetStudent invoke
2019/06/01 08:53:01 Receive message: {0 name0 {} [] 0}
2019/06/01 08:53:01 Receive message: {1 name1 {} [] 0}
2019/06/01 08:53:01 Receive message: {2 name2 {} [] 0}
2019/06/01 08:53:01 Receive message: {3 name3 {} [] 0}
2019/06/01 08:53:01 Receive message: {4 name4 {} [] 0}
2019/06/01 08:53:01 Receive message: {5 name5 {} [] 0}
2019/06/01 08:53:01 Receive message: {6 name6 {} [] 0}
2019/06/01 08:53:01 Receive message: {7 name7 {} [] 0}
2019/06/01 08:53:01 Receive message: {8 name8 {} [] 0}
2019/06/01 08:53:01 Receive message: {9 name9 {} [] 0}
```

## gRPC网关服务

用golang提供的gRPC服务端，可接受任何语言编写的客户端请求，但有时很多系统只提供了http rest请求方式，并没有开发客户端，这就需要在gRPC服务端之前部署一个http的代理服务，
这个http服务类似是gRPC的网关，这样需要与gRPC服务端交互的系统，直接以http rest方式请求grpc-gateway，gateway将对应的请求转发给gRPC服务端。

![proto](/blog/go_base/gateway.png)

要使用grpc-gateway，同样需要根据.proto文件生成对应语言的代码，通过编译工具插件protoc-gen-grpc-gateway可实现。

```go
go get -u github.com/grpc-ecosystem/grpc-gateway/protoc-gen-grpc-gateway
```

安装成功之后，protoc-gen-grpc-gateway可执行文件则移动到$GOPATH/bin目录路下，也可以手动进入源代码目录执行`go build -v`，然后copy到任何$PATH中包含的目录。 
生成grpc-gateway代码有两种方式，一种是直接在.proto文件中添加annotation，另外一种则是通过yaml配置文件进行配置，无需修改原始的proto文件，两种方式根据各自场景进行选择，
比如proto文件是公共文件，可能会在多个项目中使用，那么最好的方式就是选择基于yaml配置生成，不去修改proto文件。

由于http rest风格请求，请求参数都在请求path中，必须与gRPC中的消息结构字段一致，如下面这个代码，`{}`中的名称与Service方法中的参数对象中字段保持一致。

```go
/ Returns a specific book.
rpc GetBook(GetBookRequest) returns (Book) {
  // Client example - get the first book from the second shelf:
  //   curl http://DOMAIN_NAME/v1/shelves/2/books/1
  option (google.api.http) = { get: "/v1/shelves/{shelf}/books/{book}" };
}
...
// Request message for GetBook method.
message GetBookRequest {
  // The ID of the shelf from which to retrieve a book.
  int64 shelf = 1;
  // The ID of the book to retrieve.
  int64 book = 2;
}
```

### annotation方式

首先修改student.proto文件，给每个需要通过http rest请求的方法添加`option`，同时import相应的`annotations.proto`包。

```go
syntax = "proto3";

option go_package = "sunjinfu/api/proto";

import "google/protobuf/empty.proto";

import "google/api/annotations.proto";

message SearchRequest {
  string id = 1;
  string name = 2;
}

message SearchResponse {
  repeated Student students = 1;
}

message Student {
  int64 id = 1;
  string name = 2;
  Address address = 3;
}

message Address {
  string street = 1;
  string postcode = 2;
}

service StudentService {

  rpc GetAllStudents(google.protobuf.Empty) returns (SearchResponse) {
    option (google.api.http) = { get: "/v1/students" };
  }
  
  rpc AddStudent(Student) returns (google.protobuf.Empty) {
    option (google.api.http) = {
      post: "/v1/students"
      body: "*" 
    };
  }
  
  rpc SearchStudent(SearchRequest) returns (Student) {
    option (google.api.http) = { get: "/v1/students/ids/{id}/names/{name}" };
  }
  
  rpc GetStudent(stream SearchRequest) returns (SearchResponse) {
    option (google.api.http) = { get: "/v2/students" };
  }
}
```

proto文件添加annotation之后，再通过编译插件生成对应的代码，注意带有stream类型参数的方法映射URL是不允许有path参数的。

```bash
protoc -Ie:/grpc/include -I$GOPATH/src/github.com/grpc-ecosystem/grpc-gateway/third_party/googleapis \
-Ie:/grpc/proto --grpc-gateway_out=logtostderr=true:. student.proto
```

命令成功之后，则会在指定的包下生成对应的文件student.pb.gw.go。

### YAML配置方式

这种方式无需修改proto文件内容，只需要编写一个yaml文件student_api.yaml，在yaml文件中对需要通过http rest方式请求的方法进行配置。

```go
type: google.api.Service
config_version: 3

http: 
  rules: 
  - selector: StudentService.GetAllStudents
    get: /v1/students
  - selector: StudentService.AddStudent
    post: "/v1/students"
    body: "*"
  - selector: StudentService.SearchStudent
    get: /v1/students/ids/{id}/names/{name}
  - selector: StudentService.GetStudent
    get: /v2/students
```

执行命令生成代码，通过参数`grpc_api_configuration`指定yaml文件。

```bash
protoc -Ie:/grpc/include -I$GOPATH/src/github.com/grpc-ecosystem/grpc-gateway/third_party/googleapis \
-Ie:/grpc/proto --grpc-gateway_out=logtostderr=true,grpc_api_configuration=student_api.yaml:. student.proto
```

### 部署http网关服务

首先将生成的student.gw.pb.go文件也添加到$GOPATH/sunjinfu/api/proto包下，然后新建一个main.go文件，编写http server代码。

```go
package main

import (
	"context"
	"github.com/grpc-ecosystem/grpc-gateway/runtime"
	"google.golang.org/grpc"
	"log"
	"net/http"
	"sunjinfu/api/proto"
)


func run() error {

	endpoint := "localhost:50051"

	ctx := context.Background()
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()

	mux := runtime.NewServeMux()
	opts := []grpc.DialOption{grpc.WithInsecure()}
	err := proto.RegisterStudentServiceHandlerFromEndpoint(ctx, mux, endpoint, opts)
	if err != nil {
		log.Printf("Failed to connet to gRPC server, err: %v", err)
		return err
	}
	return http.ListenAndServe(":8080", mux)
}

func main() {
	if err := run(); err != nil {
		log.Fatal(err)
	}
}
```

http网关服务启动成功之后，通过`curl`请求8080端口服务(部分输出信息已截断)。

```bash
$ curl -X GET http://localhost:8080/v1/students
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   176  100   176    0     0  11733      0 --:--:-- --:--:-- --:--:-- 11733
{"students":[{"id":"1","name":"开发者","address":{"street":"Zh.Load","postcode":"100020"}}...}]}

$ curl -X GET http://localhost:8080/v2/students
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   176  100   176    0     0  11000      0 --:--:-- --:--:-- --:--:-- 11000
{"students":[{"id":"1","name":"开发者","address":{"street":"Zh.Load","postcode":"100020"}}...}]}


```
