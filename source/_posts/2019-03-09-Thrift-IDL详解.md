---
title: Thrift IDL详解
categories: Java
tags:
  - thrift
  - python
  - Java
abbrlink: 4ed356c2
date: 2019-02-12 15:32:32
---

### IDL

Thrift采用IDL(Interface Definition Language)来定义通用的服务接口，然后通过Thrift提供的编译器，可以将服务接口编译成不同语言编写的代码，通过这个方式来实现跨语言调用。

### 语法参考

#### 基本类型

- bool: 布尔值(true or false)
- byte: 有符号字节
- i16: 16位有符号整型
- i32: 32位有符号整型
- i64: 64位有符号整型
- double: 64位浮点型
- string: 字符串类型
- binary: byte[]数组，主要在java中使用

> 注意：Thrift不支持无符号整型，因为很多目标语言不存在无符号整型(如：java)

<!-- more -->

#### 结构体

Thrift结构体在概念上同C语言结构体类型--一种将相关属性聚集在一起的方式。在面向对象语言中，thrift结构体被转换成类。

- struct不能继承，但是可以嵌套，不能嵌套自己
- 其成员必须有明确类型
- 成员必须通过正整数进行编号，且同一个struct中的成员编号不能重复。建议不要轻易修改数字编号，修改之后如果客户端和服务端的生成代码没有同步，解析就会出现问题
- 成员分割符可以使逗号(,)或是分号(;)，而且可以混用，但是为了表述清晰，建议在定义中只使用其中一种
- 字段会有optional和required之分，如果不指定则为无类型--可以不填充该值，但是在序列化传输的时候也会序列化进去，optional是不填充则不序列化，required是必须填充也必须序列化
- 每个字段可以设置默认值
- 同一文件可以定义多个struct，也可以定义在不同文件，使用include进行引入

例子：

```c
struct Person{
    1: required string name, // 该字段必须填充
    2: optional i32 age = 20, // 该字段可以不填充，使用默认值：20
    3: i32 height // 对于基本类型，会序列化到字节数组中，解析的时候不会校验。对于对象类型，如果不为空，则序列化到自己数组中
}
```

规则：

- 如果required标识的域没有赋值，thrift将给予提示
- 如果optional标识的域没有赋值，该域将不会被序列化传输
- 如果某个optional标识的域有缺省值而用户没有重新赋值，则该域的值为缺省值
- 如果某个optional标识的域有缺省值或者用户已经重新赋值，而不设置它的__isset为true，也不会被序列化传输

#### 容器类型

thrift容器域类型密切相关，它与当前流行编程语言提供的容器类型相对应，采用java泛型风格表示。提供了3种容器类型

- list<T>: 元素类型为T的有序表，允许元素重复。对应c++的vector，java的ArrayList或者其它语言的数组
- set<T>: 元素类型为T的无序表，不允许元素重复，对应c++中的set，java中的HashSet，python中的set，php中没有set，则转换为list类型
- map<K, V>: 键类型为K，值类型为V的键值对类型，键不允许重复。对应c++中的map，Java中的HashMap，PHP中的array，Python中的dictionary

容器中元素类型可以使除了service外的任何合法thrift类型(包括结构体和异常)。

例子：

```c
struct Test {
    1: map<string, Person> personMap,
    2: set<i32> personIds,
    3: list<double> prices
}
```

#### 枚举

可以像C/C++那样定义枚举类型

- 编译器默认从0开始赋值
- 可以赋予某个常量某个整数
- 允许常量是十六进制整数
- 末尾没有分号
- 给常量赋缺省值时，使用常量的全称

例子：

```c
enum EnOpType {
  CMD_OK = 0, // (0)
  CMD_NOT_EXIT = 2000, // (2000)
  CMD_EXIT = 2001, // (2001)
  CMD_ADD = 2002 // (2002)
}
```

```c
struct StUser {
  1: required i32 userId;
  2: required string userName;
  3: optional EnOpType cmd_code = EnOpType.CMD_OK;
  4: optional string language = “english”
}
```

> 注意，不同于protocal buffer，thrift不支持枚举类嵌套，枚举常量必须是32位的正整数

#### 常量定义

thrift允许用户定义常量，复杂的类型和结构体可使用JSON形式表示，通过在变量前面使用const修饰来定义常量。

```c
const i32 INT_CONST = 1234;    // a
 
const map<string,string> MAP_CONST = {"hello": "world", "goodnight": "moon"}
```

#### 类型定义

thrift支持C/C++风格的typedef

```c
typedef i32 MyInteger   \\a
 
typedef Tweet ReTweet  \\b
```

> 注意，类型定义的末尾没有逗号或者分号；也可以使用typedef对struct类型进行重命名

#### 异常

异常在语法和功能上类似于结构体，差别是异常使用关键字exception，而且异常是继承每种语言的基础异常类。

```c
exception MyException {
    1: i32 errorCode,
    2: string message
}
```

#### 服务

服务的定义方法在语义上等同于面向对象语言中的接口，thrift编译器会产生执行这些接口的client和server stub。thrift编译器会根据选择的目标语言为server产生服务接口代码，为client产生stubs。

- 函数定义可以使用逗号或者分号标识结束
- 参数可以是基本类型或者结构体，参数是只读的（const），不可以作为返回值！！！
- 返回值可以是基本类型或者结构体，也可以是void
- service支持继承，一个service可使用extends关键字继承另一个service，但是不支持重载

```c
service HelloService {
    i32 sayInt(1:i32 param)
    string sayString(1:string param)
    bool sayBoolean(1:bool param)
    void sayVoid()
}
```

#### 命名空间

Thrift中的命名空间类似于C++中的namespace和Java中的package，它们提供了一种组织（隔离）代码的简便方式。命名空间也可以用于解决类型定义中的名字冲突。因为每种语言均有自己的命名空间定义方式（如python中有module），thrift允许开发者针对特定语言定义namespace：

```c
namespace cpp com.example.test
namespace java com.example.test
namespace py com.example.test
```

#### 注释

Thrift支持C多行风格和Java/C++单行风格。

```c
/** 
 * This is a multi-line comment. 
 * Just like in C. 
 */
 // C++/Java style single-line comments work just as well.
```

#### 包含

为了便于管理、重用和提高模块性/组织性，我们常常分割Thrift定义在不同的文件中。包含文件搜索方式与c++一样。Thrift允许文件包含其它thrift文件，用户需要使用thrift文件名作为前缀访问被包含的对象，如：

```c
include "tweet.thrift"           // a
 
struct TweetSearchResult {
	1: list<tweet.Tweet> tweets; // b
}
```

> 注意：thrift文件名要用双引号包含，末尾没有逗号或者分号

### 综合示例

entity.thrift

```c
namespace java cn.windylee.thrift.entity
namespace py course.entity

enum PhoneType{
    HOME,
    WORK,
    MOBILE,
    OTHER
}

struct Phone{
    1: i32 id,
    2: string number,
    3: PhoneType type
}

struct Person{
    1: i32 id,
    2: string firstName,
    3: string lastName,
    4: string email,
    5: list<Phone> phones
}

struct Course{
    1: i32 id,
    2: string number,
    3: string name,
    4: Person instructor,
    5: string roomNumber,
    6: list<Person> students
}
```

exception.thrift

```c
namespace java cn.windylee.thrift.exception
namespace py course.exception

exception CourseNotFound{
    1: string message
}

exception UnacceptableCourse{
    1: string message
}
```

courseService.thrift

```c
namespace java cn.windylee.thrift.service
namespace py course.service

include "Phone.thrift"
include "Exception.thrift"

service CourseService{
    list<string> getCourseInventory();
    Phone.Course getCourse(1:string courseNumber) throws (1: Exception.CourseNotFound cnf);
    void addCourse(1: Phone.Course course) throws (1: Exception.UnacceptableCourse uc);
    void deleteCourse(1:string courseNumber) throws (1:Exception.CourseNotFound cnf);
}
```

### 接口文件升级

thrift文件内容可能会随着时间变化的。如果已经存在的消息类型不再符合设计要求，比如，新的设计要在message格式中添加一个额外字段，但你仍想使用以前的thrift文件产生的处理代码。如果想要达到这个目的，只需：

- 不要修改已存在域的整数编号
- 新添加的域必须是optional的，以便格式兼容。对于一些语言，如果要为optional的字段赋值，需要特殊处理，比如对于C++语言，要将新添加字段的__isset设置为true，这样才能序列化并传输或者存储（不然optional字段被认为不存在，不会被传输或者存储）。
- 非required域可以删除，前提是它的整数编号不会被其他域使用。对于删除的字段，名字前面可添加“OBSOLETE_”以防止其他字段使用它的整数编号。

