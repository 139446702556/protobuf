语言指南
====

[定义消息类型]()  
[标量值类型]()  
[可选值和默认值]()  
[枚举]()  
[使用其他消息类型]()  
[嵌套类型]()  
[更新消息类型]()  
[拓展]()  
[Oneof 字段]()  
[Maps]()  
[Packages]()  
[定义服务]()  
[选项]()  
[生成一个类]()  

此文档将介绍如何使用protocol buffer来构造protobuf数据，包括`proto`文件的书写语法和生成数据访问类的方式。包括**proto2**版本。欲了解更多有关**proto3**的内容，请访问[Proto3语言指南]()。  

这是一个参考指南——对于许多功能，要想了解更多深层次的示例，请参考所选语言的[教程]()。  

## 定义消息类型

首先来看一个非常简单的例子。假设要定义一个查询请求格式的消息，包括一个查询字符串，所查询的感兴趣页面，每一页中符合查询要求的结果数。下面是下面是代码：  

    message SearchRequest{
    	required string query = 1;
    	optional int32 page_number = 2;
    	optional int32 result_per_page = 3;
    }  
    
`SearchRequest`消息的定义指明了三个字段（键/值对），我们希望获得的数据就包含在这样类型的消息中。每个字段都有一个名字和一个类型。

### 指明字段类型
在上面的例子中，所有的字段都是标量类型：两个整型（`page_number`和`result_per_page`）和一个字符串（`query`）。同样的，也可以为字段指定符合类型，包括枚举和其它消息类型。

### 分配标签
如你所见，消息定义中的每个字段都有一个唯一的数字标签。这些标签用来识别在消息的二进制格式中定义的字段，并且一旦使用了这些标签，就不应该再改变它们。注意从1到15的标签只需要一个字节编码，包括识别序号和字段类型（可以在[Protocol Buffer编码]()中找到更多相关信息）。从16到2047的标签需要两个字节。所以我们应该将1到15的标签用于频繁使用的消息。同样的，也应该预留一些标签用于以后可能增加的情况。

可以指定的最小标签数字是1，可以指定的最大标签数字是2^29-1（536870911）。位于19000到19999的标签不可使用。（FieldDescriptor::kFirstReservedNumber到FieldDescriptor::kLastReserNumber），这些标签是供Protocol Buffers所使用的——当你在`proto`文件中使用这个区间内的标签时，protocol buffers编译器将会给出警告。同样的，你也不能使用预先定义保留标签。


### 指定字段规则

消息字段是以下之一：
- `required`：一个好的消息格式必须要包括这个字段。
- `optional`：一个好的消息格式可以有零个或一个此字段。（但不可多于一个）
- `repeated`：此字段可以重复任意多次（包括零次），并且值的顺序会被记录。

因为历史性的原因，标量数字类型的`repeated`字段的编码效率并不如想象中的那样好。在今后的代码中应该使用特殊的选项`[packed=true]`来更高效地编码。比如：

    repeated int32 samples = 4 [packed=true];

在[Protocol Buffer编码]()中可以获取更多有关于`packed`的信息。

**Required是永久的**：在将字段声明成**required**的时候，读者应该非常小心。如果因为某些原因读者希望停止写和发送required字段，那么在将其转换为optional字段时，将产生一些严重的问题——在旧式读取中这样的字段可能会被认为是不完整的，从而无意中拒绝或删除此字段。读者应该考虑为buffers编写特定的、针对应用程序的自定义验证过程。谷歌的一些工程师的出这样的结论：使用**required**弊大于利；他们更偏向于只用**optional**和**repeated**字段。但是，这样的观点并不是广泛的。

### 加入更多消息类型

在一个`.proto`文件中可以定义多个消息类型。这是很有用的，比如说当我们想针对`SearchRequest`做出回应的时候，可以定义一个`SearchResponse`消息类型，可以在`.proto`文件中这么写：

    message SearchRequest {
      required string query = 1;
      optional int32 page_number = 2;
      optional int32 result_per_page = 3;
    }
    
    message SearchResponse {
     ...
    }

### 添加注释

使用C/C++风格的写法来添加注释。

    message SearchRequest {
      required string query = 1;
      optional int32 page_number = 2;// Which page number do we want?
      optional int32 result_per_page = 3;// Number of results to return per page.
    }

### 保留字段

如果通过删除或注释整个字段的方式来更新一个消息类型的话，此代码的下一任开发者在创建其它的消息类型时，可以重新使用标签。但是当加载相同的`.proto`的旧版本时将导致一些问题，包括数据冲突，潜在BUG，等等。Json解析同样存在这样的问题，一个确保避免此问题的方法是指定你所删除的字段为`reserved`。如果下一任开发者尝试使用这些字段时，protocol buffer编译器将给出警告。

    message Foo {
      reserved 2, 15, 9 to 11;
      reserved "foo", "bar";
    }

请注意，不要在同一个保留字段声明中混淆字段名和标签序号。

### **.proto**文件生成了什么？

当使用protocol buffer编译器编译一个`.proto`时，编译器将根据所选编程语言生成可以用于处理消息的代码。包括`set`和`get`字段值，将消息序列化成输出流，从输入流解析消息等。

- 对于**C++**语言来说，编译器生成`.h`和`.cc`文件，并针对每个消息类型生成一个类。
- 对于**Java**语言来说，编译器生成`.java`文件，并针对每个消息类型生成一个类，以及一个用于产生单例模式的`Builder`类。
- 对于**Python**语言来说就有点不同了——Python编译器针对每个消息类型生成一个静态描述符模块，然后使用meta类在运行时创建必要的Python数据访问类。
- 对于**Go**语言来说，编译器生成`.pb`和`.go`文件。

可以根据所选择语言来浏览更多关于API的使用方法。有关更多API的细节，请访问[API参考]()。

## 标量值类型

一个标量消息字段可以是以下的类型——表格列出了`.proto`文件指明的类型，对应的类型由相应的类自动生成：
![](./Image/标量类型列表.png)

