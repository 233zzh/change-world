# 什么是 RPC

RPC（Remote Procedure Call）是一种远程过程调用协议，它允许计算机程序通过网络请求远程计算机上的服务，而无需了解底层网络技术的细节。RPC 的概念是让程序像调用本地函数一样调用远程服务器上的函数，而不需要显式编写通信细节[^1](https://en.wikipedia.org/wiki/Remote_procedure_call)。

在分布式计算中，RPC 可以使不同的计算机程序在不同的地址空间中执行过程，而编程时可以像调用本地过程一样编写代码，而无需显式编写通信细节[^1](https://en.wikipedia.org/wiki/Remote_procedure_call)。

举个例子，假设有两台不同的机器，分别部署了服务 A 和服务 B。如果服务 A 想要调用服务 B 中的某个方法，可以使用 RPC 协议，通过网络请求服务 B 上的方法，而不需要了解底层网络技术的细节[^3](https://www.zhihu.com/question/25536695?utm_id=0)。

简而言之，RPC 是一种让程序可以在不同计算机上调用远程服务的协议，使得分布式系统中的不同组件能够无缝通信和交互[^2](https://zhuanlan.zhihu.com/p/187560185?utm_id=0)。

总结：**RPC 协议的主要目的是做到不同服务间调用方法像同一服务间调用本地方法一样**
**程序可以像调用本地方法一样调用远程方法**

# RPC 的原理是什么?

在 RPC 中，发起调用的程序称为客户端，接收调用的程序称为服务器。客户端发送一个请求给服务器，指定要执行的过程或函数以及必要的参数。服务器然后执行所请求的过程，并将结果发送回客户端。这使得客户端程序可以像调用本地函数或过程一样调用远程服务器上的函数或过程。

RPC（Remote Procedure Call，远程过程调用）的原理是通过网络实现分布式系统中的通信和调用。它的基本原理可以概括为以下几个步骤：

1. 定义接口：首先需要定义远程服务的接口，包括可调用的方法和参数类型。这个接口定义可以使用 IDL（接口定义语言）来描述，例如使用 Protocol Buffers、Thrift 或 gRPC 等。
2. 代理生成：在客户端和服务器端分别生成代理代码。代理代码用于封装和解析请求和响应的数据，以及进行网络通信。生成代理代码的方式可以是手动编写，也可以使用 IDL 工具根据接口定义自动生成。
3. 服务注册与发现：客户端和服务器端需要进行服务注册与发现，以便能够找到对应的服务。这可以通过使用服务注册中心（如 ZooKeeper、Consul 等）或者配置中心来实现。
4. 客户端调用：客户端通过调用本地的代理代码，将方法名和参数传递给代理。代理将这些信息封装成一个请求，并通过网络发送给服务器。通常情况下，客户端通过网络协议（如 HTTP、TCP/IP 等）将请求发送给服务器。
5. 服务器处理：服务器接收到请求后，代理代码将请求解析出方法名和参数，并调用相应的方法进行处理。服务器端需要根据接口定义的方法名，定位到对应的实现代码，并将请求的参数传递给实现代码进行处理。
6. 服务器响应：服务器执行完方法后，将结果封装成响应，并通过网络发送给客户端。服务器端通过网络协议将响应发送给客户端。
7. 客户端接收：客户端接收到响应后，代理代码将响应解析出结果，并返回给调用者。客户端将响应的结果返回给调用方，完成整个远程调用过程。

整个 RPC 过程中，代理代码在客户端和服务器端起到了桥接的作用，负责封装和解析数据，以及处理网络通信。通过使用 RPC，可以使得远程调用像本地调用一样简单，提高了分布式系统的开发效率和可维护性。

需要注意的是，

1. RPC 的实现可以有多种协议和传输方式，如使用 HTTP 协议和 JSON 格式进行通信，或使用 TCP/IP 协议和二进制格式进行通信。具体的实现方式可以根据需求和场景进行选择。
2. 不同的 RPC 框架在实现细节上可能有所不同，但基本的原理和流程是相似的。具体的实现方式可以根据需求和场景进行选择，常见的 RPC 框架有 gRPC、Apache Dubbo、Spring Cloud 等。

# PB 协议（ Protocol Buffers、Protobuf）

Protobuf 是一种数据序列化格式，它可以将结构化数据转换为二进制格式，以便在网络传输或存储时使用。以下是一些关于 Protobuf 的信息和相关链接：

1. Protobuf 是由 Google 开发的，最初用于内部 RPC 通信协议和数据存储格式。
2. Protobuf 使用.proto 文件定义数据结构和消息格式，然后使用特定的编译器生成用于不同平台、不同编程语言的代码，以便在不同平台、不同的编程语言中使用。这样，你就可以在不同的平台上使用不同的编程语言来读取和写入这些结构化数据。
3. Protobuf 具有高效的编码和解码速度，以及较小的数据体积，比 XML 和 JSON 等其他序列化格式更加高效。
4. Protobuf 支持多种编程语言，包括 C++、Java、Python 等。
5. 官方网站：https://developers.google.com/protocol-buffers
6. GitHub 仓库：https://github.com/protocolbuffers/protobuf

以下是一个使用 Protobuf 的示例代码，使用 Java 语言：

首先，需要定义一个.proto 文件，例如 person.proto：
定义了一个名为 Person 的消息类型，包含了 name、age 和 hobbies 三个字段。`syntax = "proto3"`  表示使用 Protocol Buffers 的第三个版本。

```protobuf
syntax = "proto3";

message Person {
  string name = 1;
  int32 age = 2;
  repeated string hobbies = 3;
}
```

然后，使用 protoc 编译器生成 Java 代码：

```
protoc --java_out=. person.proto
```

接下来，可以在 Java 代码中使用生成的类来序列化和反序列化数据：

```java
import com.example.Person;

public class Main {
  public static void main(String[] args) throws Exception {
    Person.Person.Builder builder = Person.Person.newBuilder();
    builder.setName("John");
    builder.setAge(25);
    builder.addHobbies("Reading");
    builder.addHobbies("Gaming");

    Person.Person person = builder.build();

    // 序列化为字节数组
    byte[] data = person.toByteArray();

    // 反序列化
    Person.Person parsedPerson = Person.Person.parseFrom(data);

    System.out.println("Name: " + parsedPerson.getName());
    System.out.println("Age: " + parsedPerson.getAge());
    System.out.println("Hobbies: " + parsedPerson.getHobbiesList());
  }
}
```

这只是一个简单的示例，你可以根据自己的需求定义更复杂的消息结构和使用不同的编程语言进行开发。请注意，具体的代码实现可能因编程语言而异。你可以根据你选择的编程语言和 PB 的库来查找相关的代码示例和文档

# 下一篇：《既然有了 HTTP 协议，为什么还要有 RPC ？》
