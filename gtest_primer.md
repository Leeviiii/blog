[原始文档](https://github.com/google/googletest/blob/master/googletest/docs/primer.md)
[TOC]
## 介绍:为何选择googletest?
googletest可以帮助你写出更好的C++测试代码。

googletest是google技术团队开发的一款测试框架，下面简称gtest。gtest支持Linux, Windows以及Mac。gtest支持任何类型的测试，而不仅仅是unit test。

google团队认为好的test应该满足以下性质，并且gtest都一一满足：

1. Tests应该是独立且可重复的。独立指的是多个tests之间互不影响，一个test失败或者成功不应该受其他test的影响。gtest将test进行隔离，每一个test对应一个隔离的object。当一个test失败，gtest允许用户单独快速debug这个test。
2. Tests应该是组织良好并且有一定结构的。gtest可以将一些相关的test分组，这一组test可以共享一些数据以及行为
3. Tests应该是可移植的。gtest支持多个平台，具有可移植性
4. 当一个test失败的时候，gtest会提供很多的辅助信息帮助开发者快速定位问题。
5. 好的测试框架应该将开发者从烦杂的测试工作抽离出来，而只关注测试内容本身。gtest只需要开发者写好测试内容即可，不需要枚举测例进行注册等操作
6. gtest是很快的，测例之间可以共享set-up/tear-down

gtest是基于xUnit架构的，因此如果使用过JUnit或者PyUnit，会上手很快，没用过也没关系。
## 基本概念
在使用gtest过程中，开发者需要写一些断言。gtest定义断言的结果可以是success, nonfatal failure或者 fatal failure三种。如果fatal failure发生了，那么gtest会直接终止当前的函数，否则会继续执行。

开发者应该将众多的测例进行分组，相关的测试放在一起进行组织。如果他们之间需要共享一些数据或者行为，可以将这组test放入一个test fixture class(后面介绍)。

## 断言
gtest提供了丰富的断言宏定义帮助开发者方便的写自己的测试。当一个断言失败时，gtest会打印出文件名以及行号以及一些相关的失败信息。开发者也可以定制自己的失败信息。

gtest提供两种版本的断言`ASSERT_*`以及`EXPECT_*`。当一个断言失败的时候，`ASSERT_*`开头的断言会产生fatal failure，`EXPECT_*`开头的断言则会产生nonfatal failure。具体选择哪种断言要根据具体的测试逻辑，如果二者即可，那么建议选择`EXPECT_*`。

可以通过操作符`<<` 来定制化失败信息。
```c++
ASSERT_EQ(x.size(), y.size()) << "Vectors x and y are of unequal length";
 
for (int i = 0; i < x.size(); ++i) {
  EXPECT_EQ(x[i], y[i]) << "Vectors x and y differ at index " << i;
}
```
### 基本断言
true/false断言

Fatal assertion            | Nonfatal assertion         | Verifies
-------------------------- | -------------------------- | --------------------
`ASSERT_TRUE(condition);`  | `EXPECT_TRUE(condition);`  | `condition` is true
`ASSERT_FALSE(condition);` | `EXPECT_FALSE(condition);` | `condition` is false
### 比较断言

Fatal assertion          | Nonfatal assertion       | Verifies
------------------------ | ------------------------ | --------------
`ASSERT_EQ(val1, val2);` | `EXPECT_EQ(val1, val2);` | `val1 == val2`
`ASSERT_NE(val1, val2);` | `EXPECT_NE(val1, val2);` | `val1 != val2`
`ASSERT_LT(val1, val2);` | `EXPECT_LT(val1, val2);` | `val1 < val2`
`ASSERT_LE(val1, val2);` | `EXPECT_LE(val1, val2);` | `val1 <= val2`
`ASSERT_GT(val1, val2);` | `EXPECT_GT(val1, val2);` | `val1 > val2`
`ASSERT_GE(val1, val2);` | `EXPECT_GE(val1, val2);` | `val1 >= val2`

这些断言如果应用于用户自定义的类型，那么需要重载对应的操作符(如 `==`, `<`, 等)。重载操作符可以参考 [C++ Style Guide](https://google.github.io/styleguide/cppguide.html#Operator_Overloading)。开发者可以通过断言 `ASSERT_TRUE()` 或者 `EXPECT_TRUE()` 来判断两个object或者自定义的类型是否相等。

表达`ASSERT_EQ(actual, expected)` 比`ASSERT_TRUE(actual == expected)`更好，前者可以告诉我们更多信息，哪个值是实际值，哪个值是期望值。

`ASSERT_EQ()`会比较两个指针是否相等。如果使用两个C类型的字符串，断言`ASSERT_EQ()`判断指针的地址是否相同，而不是字符串的内容是否相同。可以使用断言`ASSERT_STREQ()`来比较两个C类型(e.g. `const char*`)的字符串的内容是否相同。断言`ASSERT_STREQ(c_string, NULL)`可以判断一个C类型的字符串是否为NULL，如果c++11支持，也可以使用`ASSERT_EQ(c_string, nullptr)`来判断一个C类型的字符串是否为NULL。如果比较两个`string`对象，应该使用`ASSERT_EQ`.

断言 `*_EQ(ptr, nullptr)` 以及 `*_NE(ptr, nullptr)`优于`*_EQ(ptr, NULL)`以及`*_NE(ptr, NULL)`。这是因为`nullptr`是一种类型，而`NULL`则不是。

### 字符串比较
| Fatal assertion                 | Nonfatal assertion              | Verifies                                                 |
| ------------------------------- | ------------------------------- | -------------------------------------------------------- |
| `ASSERT_STREQ(str1, str2);`     | `EXPECT_STREQ(str1, str2);`     | the two C strings have the same content                  |
| `ASSERT_STRNE(str1, str2);`     | `EXPECT_STRNE(str1, str2);`     | the two C strings have different contents                |
| `ASSERT_STRCASEEQ(str1, str2);` | `EXPECT_STRCASEEQ(str1, str2);` | the two C strings have the same content, ignoring case   |
| `ASSERT_STRCASENE(str1, str2);` | `EXPECT_STRCASENE(str1, str2);` | the two C strings have different contents, ignoring case |
## 简单的小例子

通过三步创建一个简单的测试:
 
1.  使用`TEST()`宏定义一个测试，这个宏是一个没有返回值的C++函数
1.  在这个函数内部写自己的测试逻辑以及断言
1.  测试结果依赖断言的返回，如果每一个断言都返回成功，则整个测试成功，否则测试失败。

```c++
TEST(TestCaseName, TestName1) {
  ... test body ...
}
TEST(TestCaseName, TestName2) {
  ... test body ...
}
```
gtest会将相同TestCaseName的测例自动分为一组。TestCaseName以及TestName不可以包含下划线(`_`)。gtest表示一个测试的完整名字使用TestCaseName.TestName来表示。

下面举一个简单的例子
```c++
int Factorial(int n);  // Returns the factorial of n
```
测试函数Factorial的测例如下
```c++
// Tests factorial of 0.
TEST(FactorialTest, HandlesZeroInput) {
  EXPECT_EQ(Factorial(0), 1);
}
 
// Tests factorial of positive numbers.
TEST(FactorialTest, HandlesPositiveInput) {
  EXPECT_EQ(Factorial(1), 1);
  EXPECT_EQ(Factorial(2), 2);
  EXPECT_EQ(Factorial(3), 6);
  EXPECT_EQ(Factorial(8), 40320);
}
```
上面的两个测试都是对函数Factorial的测试，因此他们的TestCaseName相同，两个测试被分成了一组。我们在写自己的测试的时候也要遵循这种组织方式。

## Test Fixtures: 多个测试之间使用相同的数据

*test fixture*可以帮助我们实现多个测试共用一份相同或者相似的数据。

通过下面的方法创建一个*fixture*

1.  定义一个`::testing::Test`的子类,类内部的共享数据定义为 `protected:`，gtest会以子类的方式来使用这些数据。
1.  在类中定义需要共享的数据
1.  如果需要，可以实现一个构造器或者重写函数`SetUp()`进行数据准备工作。
1.  如果需要，可以实现一个析构函数或者`TearDown()`进行数据清理工作。
1.  如果需要，可以定义一些共享的方法，多个测试可以一起使用。

使用gtest的*fixture*需要宏`TEST_F()`而不再是`TEST()`，并且`TEST_F()`的第一个参数TestCaseName需要上面第一步中定义的`::testing::Test`的子类的名字。

每一个通过`TEST_F()`定义的测试，gtest会在运行时创建一个干净的测试fixture，然后立即调用函数 `SetUp()`进行初始化，然后运行测试内容，然后调用函数`TearDown()`进行清理工作并且删除这个测试fixture。gtest会在一个相同的test case(多个TEST_F的第一个参数相同即为相同的test case)中创建不同的fixture对象，gtest会在创建下一个fixture对象前删除当前的fixture对象。对于多个测试，gtest不会重复使用相同的fixture，因此对于一个fixture的改动不会影响其他的fixture。

下面测试一个FIFO的队列，接口如下
```c++
template <typename E>  // E is the element type.
class Queue {
 public:
  Queue();
  void Enqueue(const E& element);
  E* Dequeue();  // Returns NULL if the queue is empty.
  size_t size() const;
  ...
};
```
首先我们定义一个fixture，通常应该是Test结尾。
```c++
class QueueTest : public ::testing::Test {
 protected:
  void SetUp() override {
     q1_.Enqueue(1);
     q2_.Enqueue(2);
     q2_.Enqueue(3);
  }
 
  // void TearDown() override {}
 
  Queue<int> q0_;
  Queue<int> q1_;
  Queue<int> q2_;
};
```
测试如下
```c++
TEST_F(QueueTest, IsEmptyInitially) {
  EXPECT_EQ(q0_.size(), 0);
}
 
TEST_F(QueueTest, DequeueWorks) {
  int* n = q0_.Dequeue();
  EXPECT_EQ(n, nullptr);
 
  n = q1_.Dequeue();
  ASSERT_NE(n, nullptr);
  EXPECT_EQ(*n, 1);
  EXPECT_EQ(q1_.size(), 0);
  delete n;
 
  n = q2_.Dequeue();
  ASSERT_NE(n, nullptr);
  EXPECT_EQ(*n, 2);
  EXPECT_EQ(q2_.size(), 1);
  delete n;
}
```
上面的例子中使用了`ASSERT_*`以及`EXPECT_*`断言。原则上，如果使用`EXPECT_*`表示当断言失败的时候希望测试继续执行以发现更多的错误，而使用`ASSERT_*`则表示当断言失败的时候函数立即停止执行。 上面对于`Dequeue`测试的第二个断言`ASSERT_NE(nullptr, n)`一旦失败，则后面继续执行会发生段错误，因此使用`ASSERT_*`。

gtest会按照下面的过程来执行上面的测试：
1.  gtest创建一个`QueueTest`对象 ( 记为`t1` )，
1.  `t1.SetUp()`初始化`t1` ，
1.  在`t1`上执行( `IsEmptyInitially` )测试 ，
1.  测试完成后，调用`t1.TearDown()`进行清理 ，
1.  `t1`被删除，
1.  上面的过程在另一个`QueueTest`对象上重复，用于进行`DequeueWorks`测试( gtest隔离性的一个体现 )。
## 启动gtest

`TEST()`以及`TEST_F()`会隐式的将测试注册到gtest上。 通过调用`RUN_ALL_TESTS()`运行所有的测试。所有测试成功，函数返回0，否则返回1。
 
> 不可以忽略`RUN_ALL_TESTS()`的返回值，否则会有编译错误。
> 
> `RUN_ALL_TESTS()`只能被调用一次，不可以被调用多次。 

### 写main()函数

```c++
#include "this/package/foo.h"
#include "gtest/gtest.h"
 
namespace {
 
// The fixture for testing class Foo.
class FooTest : public ::testing::Test {
 protected:
  // You can remove any or all of the following functions if its body
  // is empty.
 
  FooTest() {
     // You can do set-up work for each test here.
  }
 
  ~FooTest() override {
     // You can do clean-up work that doesn't throw exceptions here.
  }
 
  // If the constructor and destructor are not enough for setting up
  // and cleaning up each test, you can define the following methods:
 
  void SetUp() override {
     // Code here will be called immediately after the constructor (right
     // before each test).
  }
 
  void TearDown() override {
     // Code here will be called immediately after each test (right
     // before the destructor).
  }
 
  // Objects declared here can be used by all tests in the test case for Foo.
};
 
// Tests that the Foo::Bar() method does Abc.
TEST_F(FooTest, MethodBarDoesAbc) {
  const std::string input_filepath = "this/package/testdata/myinputfile.dat";
  const std::string output_filepath = "this/package/testdata/myoutputfile.dat";
  Foo f;
  EXPECT_EQ(f.Bar(input_filepath, output_filepath), 0);
}
 
// Tests that Foo does Xyz.
TEST_F(FooTest, DoesXyz) {
  // Exercises the Xyz feature of Foo.
}
 
}  // namespace
 
int main(int argc, char **argv) {
  ::testing::InitGoogleTest(&argc, argv);
  return RUN_ALL_TESTS();
}
```
函数`::testing::InitGoogleTest()`会解析命令行参数，gtest可以通过命令行参数支持一些高级操作。可以在[AdvancedGuide](advanced.md)文档进行查看。这个函数必须要在`RUN_ALL_TESTS()`之前调用。

## 编译

1.8.x Release - 1.8.x是gtest的最新发布版本，编译需要C++11支持。[下载地址](https://github.com/google/googletest/releases/tag/release-1.8.1)。下载解压进入googletest目录。

使用cmake进行编译
```
mkdir build
cd build
cmake -DBUILD_SHARED_LIBS=on ..
make -j8
```
然后将编译好的libgtest_main.so  libgtest.so拷贝到ctr_feature的deps相应目录下，gtest的.h文件也需要一并拷贝过去。
## 其他
gtest还有很多高级应用，可自行查阅文档。

