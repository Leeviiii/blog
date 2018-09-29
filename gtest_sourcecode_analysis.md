gtest的源码足够精简，gtest使用了c++11可以帮助我学习很多c++的基础特性
## 例子
```c++
#include "gtest/gtest.h"

TEST(FooTest, Demo) {
  EXPECT_EQ(1, 1);
}
int main(int argc, char **argv) {
  ::testing::InitGoogleTest(&argc, argv);
  return RUN_ALL_TESTS();
}

```
上面的代码我们进行一次预编译可以得到下面这样的片段
```c++
# 4 "src/gtest_main.cpp" 2
class FooTest_Demo_Test : public  { 
  public: 
    FooTest_Demo_Test() {} 
  private: 
    virtual void TestBody(); 
    static ::testing::TestInfo* const test_info_ __attribute__ ((unused)); 
    FooTest_Demo_Test(FooTest_Demo_Test const &) ;
    void operator=(FooTest_Demo_Test const &) ;
};
::testing::TestInfo* const FooTest_Demo_Test ::test_info_ = 
      ::testing::internal::MakeAndRegisterTestInfo( "FooTest", "Demo",
                                                     __null, __null, 
                                                     ::testing::internal::CodeLocation("src/gtest_main.cpp", 5), 
                                                     (::testing::internal::GetTestTypeId()), 
                                                     ::testing::Test::SetUpTestCase, 
                                                     ::testing::Test::TearDownTestCase, 
                                                     new ::testing::internal::TestFactoryImpl< FooTest_Demo_Test>);
void FooTest_Demo_Test::TestBody() {
  switch (0) 
  case 0: 
  default: 
    if (const ::testing::AssertionResult gtest_ar = (::testing::internal:: EqHelper<(sizeof(::testing::internal::IsNullLiteralHelper(1)) == 1)>::Compare("1", "1", 1, 1))) ; 
    else ::testing::internal::AssertHelper(::testing::TestPartResult::kNonFatalFailure, "src/gtest_main.cpp", 6, gtest_ar.failure_message()) = ::testing::Message();
}
```
下面一点点分析代码。
## 基础类介绍
### scoped_ptr
`scoped_ptr`就是对指针进行了简单的封装。
```c++
// This implementation of scoped_ptr is PARTIAL - it only contains
// enough stuff to satisfy Google Test's need.
template <typename T>
class scoped_ptr {
 public:    
  typedef T element_type;
            
  explicit scoped_ptr(T* p = NULL) : ptr_(p) {}
  ~scoped_ptr() { reset(); }
            
  T& operator*() const { return *ptr_; }
  T* operator->() const { return ptr_; }
  T* get() const { return ptr_; }
            
  T* release() {
    T* const ptr = ptr_;
    ptr_ = NULL;
    return ptr; 
  }         
            
  void reset(T* p = NULL) {
    if (p != ptr_) {
      if (IsTrue(sizeof(T) > 0)) {  // Makes sure T is a complete type.
        delete ptr_;
      }     
      ptr_ = p; 
    }       
  }         
            
  friend void swap(scoped_ptr& a, scoped_ptr& b) { 
    using std::swap;
    swap(a.ptr_, b.ptr_);
  }         
            
 private:   
  T* ptr_;  
            
  GTEST_DISALLOW_COPY_AND_ASSIGN_(scoped_ptr);
  // 上面这个宏展开后为
  // scoped_ptr(scoped_ptr const &) = delete;
  // void operator=(scoped_ptr const &) = delete;
};          
```
c++相关
1. `explicit`关键字用来修饰构造函数，表示禁止进行隐式类型转换。
1. `friend`可以是友元函数或者友元类，友元的意思是可以访问类的private以及protected内容
1. `GTEST_DISALLOW_COPY_AND_ASSIGN_`使用了C++11的`delete`特性，禁止了赋值以及拷贝。C++编译器会默认给我们生成很多的默认行为，如默认的构造函数，默认的赋值操作，默认的拷贝等等，使用delete可以禁止这些行为；[C++11还提供了`default`关键字](https://www.ibm.com/developerworks/cn/aix/library/1212_lufang_c11new/index.html),用于生成默认的函数体，效率也会比手动写的要高(TODO 为啥呢？？)
### AssertionResult
`AssertionResult`就是我们在使用gtest的断言的过程中，断言返回的结果了，gtest依赖这个类来判断测试是否成功还是失败以及输出辅助信息。
```c++
class GTEST_API_ AssertionResult {
 public:
  // Returns true iff the assertion succeeded.
  operator bool() const { return success_; }  // NOLINT
  const char* message() const {
    return message_.get() != NULL ?  message_->c_str() : "";
  }
  // Streams a custom failure message into this object.
  template <typename T> AssertionResult& operator<<(const T& value) {
    AppendMessage(Message() << value);
    return *this;
  }
 
  // Allows streaming basic output manipulators such as endl or flush into
  // this object.
  AssertionResult& operator<<(
      ::std::ostream& (*basic_manipulator)(::std::ostream& stream)) {
    AppendMessage(Message() << basic_manipulator);
    return *this;
  }
 private:
  // Appends the contents of message to message_.
  void AppendMessage(const Message& a_message) {
    if (message_.get() == NULL)
      message_.reset(new ::std::string);
    message_->append(a_message.GetString().c_str());
  }
  
  // Swap the contents of this AssertionResult with other.
  void swap(AssertionResult& other);
  
  // Stores result of the assertion predicate.
  bool success_;
  // Stores the message describing the condition in case the expectation
  // construct is not satisfied with the predicate's outcome.
  // Referenced via a pointer to avoid taking too much stack frame space
  // with test assertions.
  internal::scoped_ptr< ::std::string> message_;
};

// Makes a successful assertion result.  
AssertionResult AssertionSuccess() {
  return AssertionResult(true);  
}
// Makes a failed assertion result.      
AssertionResult AssertionFailure() {
  return AssertionResult(false);
}
// Makes a failed assertion result with the given failure message.
// Deprecated; use AssertionFailure() << msg.
AssertionResult AssertionFailure(const Message& msg) {
  return AssertionFailure() << message;
}
```
c++相关
1. 重载操作符`bool()`可以在if(object)下使用。[更多参考](https://www.artima.com/cppsource/safebool.html)
1. 操作符`<<`的重载应该涉及到了c++ stream的一些知识 先作为TODO 放在这吧

如果想要产生自己的`AssertionResult`需要调用函数`AssertionSuccess()`或者`AssertionFailure()`进行创建，gtest不建议自己主动创建。
### 宏GTEST_IS_NULL_LITERAL_(x)
```c++
class Secret;
char IsNullLiteralHelper(Secret* p);  
char (&IsNullLiteralHelper(...))[2];  // NOLINT 
# define GTEST_IS_NULL_LITERAL_(x) \ 
   (sizeof(::testing::internal::IsNullLiteralHelper(x)) == 1) 
```
上面这段代码困惑了我很久，宏定义`宏GTEST_IS_NULL_LITERAL_(x)`是gtest的一个基础宏，用来判断x是否为null pointer literal(NULL or any 0-valued compile-time integral constant)。从上面的行为推荐应该是如果传进来的x为NULL或者nullptr则IsNullLiteralHelper(x)会被编译器匹配到第一个函数声明，返回char因此sizeof后值为1(sizeof一个函数，是看返回这的sizeof)；而如果不是NULL的其他值，则不可以隐式转换为`Secret *`类型，则会被编译器匹配到第二个函数，返回char[2] ，sizeof的值为2，以此来判断是否为NULL。
### EqHelper
```c++
typedef __int64 BiggestInt; 
template <bool lhs_is_null_literal>
class EqHelper {
 public:
  // This templatized version is for the general case.
  template <typename T1, typename T2>
  static AssertionResult Compare(const char* lhs_expression,
                                 const char* rhs_expression,
                                 const T1& lhs, 
                                 const T2& rhs) {
    return CmpHelperEQ(lhs_expression, rhs_expression, lhs, rhs);
  }
  static AssertionResult Compare(const char* lhs_expression,
                                 const char* rhs_expression,
                                 BiggestInt lhs, 
                                 BiggestInt rhs) {
    return CmpHelperEQ(lhs_expression, rhs_expression, lhs, rhs);
  }
};

```
`EqHelper`用来辅助`{ASSERT|EXPECT}_EQ`的实现。如果ASSERT_EQ()的第一个参数是null pointer literal，则模板参数lhs_is_null_literal为true。上面的默认实现是lhs_is_null_literal为false的情况。第一个模板函数用来处理一般的情况，第二个BiggestInt的重载函数用来处理匿名枚举的情况(TODO)。

函数CmpHelperEQ的实现
```c++
//这个重载用来实现int or  enum类型的比较
AssertionResult CmpHelperEQ(const char* lhs_expression, 
                         const char* rhs_expression, 
                         BiggestInt lhs, 
                          BiggestInt rhs) {
  if (lhs == rhs) { 
    return AssertionSuccess();
  }
  return EqFailure(lhs_expression,rhs_expression,   
                 FormatForComparisonFailureMessage(lhs, rhs),
                 FormatForComparisonFailureMessage(rhs, lhs),
                 false);
// The helper function for {ASSERT|EXPECT}_EQ. 
template <typename T1, typename T2>
AssertionResult CmpHelperEQ(const char* lhs_expression,
                            const char* rhs_expression,                                               
                            const T1& lhs,
                            const T2& rhs) {                                                          
  if (lhs == rhs) {
    return AssertionSuccess();                                                                        
  }                         
  return CmpHelperEQFailure(lhs_expression, rhs_expression, lhs, rhs);                             
}                  
          
```
函数CmpHelperEQFailure式对函数EqFailure的一层简单封装，二者的作用都是输出错误信息。就不给出具体代码了。可以看出，如果比较我们自己定义的object，需要重载操作符=。
## 测试框架
### 函数MakeAndRegisterTestInfo
```c++
// Creates a new TestInfo object and registers it with Google Test;
// returns the created object.
//
// Arguments:
//
//   test_case_name:   name of the test case
//   name:             name of the test
//   type_param:       the name of the test's type parameter, or NULL if
//                     this is not a typed or a type-parameterized test.
//   value_param:      text representation of the test's value parameter,
//                     or NULL if this is not a value-parameterized test.
//   code_location:    code location where the test is defined
//   fixture_class_id: ID of the test fixture class
//   set_up_tc:        pointer to the function that sets up the test case
//   tear_down_tc:     pointer to the function that tears down the test case
//   factory:          pointer to the factory that creates a test object.
//                     The newly created TestInfo instance will assume
//                     ownership of the factory object.
TestInfo* MakeAndRegisterTestInfo(
    const char* test_case_name,
    const char* name,
    const char* type_param,
    const char* value_param,
    CodeLocation code_location,
    TypeId fixture_class_id,
    SetUpTestCaseFunc set_up_tc,
    TearDownTestCaseFunc tear_down_tc,
    TestFactoryBase* factory) {
  TestInfo* const test_info =
      new TestInfo(test_case_name, name, type_param, value_param,
                   code_location, fixture_class_id, factory);
  GetUnitTestImpl()->AddTestInfo(set_up_tc, tear_down_tc, test_info);
  return test_info;
}

// Creates an empty UnitTest.
UnitTest::UnitTest() {                                                                                                                                                                                      
  impl_ = new internal::UnitTestImpl(this);
}
```
这个函数名字很好例子就是制造一个测试并且注册给gtest。这个函数接收九个参数
1. test_case_name也就是我们TEST宏传进来的第一个参数，表示test_case名字
1. name就是TEST宏的第二个参数test的名字，
1. type_param以及value_param是进行类型测试的两个参数，暂时忽略
1. code_location是我们测试代码的一些信息，用于测试失败的时候输出的
1. fixture_class_id
1. set_up_tc与tear_down_tc默认为Test类的辅助函数，用于测试前的初始化以及测试后的清理工作，是两个回调函数
1. factory是一个工厂类用来创建TestInfo的

`GetUnitTestImpl`函数返回一个接口，这个接口全局唯一，位于类`UnitTest`中，UnitTest是一个单例，因此一旦初始化，接口就不会再变了。

函数AddTestInfo会调用GetTestCase函数，函数GetTestCase会根据test_case_name一个存在的TestCase,如果不存在则创建一个新的TestCase返回，然后调用TestCase的AddTestInfo函数，将一个具体的test添加进去。

全局的唯一的UnitTestImpl中存储了一个TestCase的列表，而每一个TestCase又存储了一个具体的Test，于是这样就注册了一个测试。每一个具体的Test是一个TestInfo结构.而具体的Test都是`::testing::Test`的一个子类。

### RUN_ALL_TESTS
```c++
inline int RUN_ALL_TESTS() {
  return ::testing::UnitTest::GetInstance()->Run();
}
```
UnitTest里面的Run实际上是调用internal::UnitTestImpl::RunAllTests
```c++
// Runs each test case if there is at least one test to run.
    if (has_tests_to_run) {
      
      if (!Test::HasFatalFailure()) {
        for (int test_index = 0; test_index < total_test_case_count();
             test_index++) {
          GetMutableTestCase(test_index)->Run();                               }              
      }                
     
    }                  
```
GetMutableTestCase会取到每一个TestCase实例进行遍历运行
```c++
// Runs every test in this TestCase.
void TestCase::Run() {
  
  for (int i = 0; i < total_test_count(); i++) {
    GetMutableTestInfo(i)->Run();
  }
  
}

```
TestCase的Run会遍历每一个TestInfo 
```c++
// Creates the test object. 
  Test* const test = internal::HandleExceptionsInMethodIfSupported(
      factory_, &internal::TestFactoryBase::CreateTest,
      "the test fixture's constructor");    
               
  // Runs the test if the constructor didn't generate a fatal failure.
  // Note that the object will not be null  
  if (!Test::HasFatalFailure()) {           
    // This doesn't throw as all user code that can throw are wrapped into
    // exception handling code.
    test->Run();
  }                                                                 
```
TestInfo的Run会先创建一个Test对象，然后调用test的Run
```c++
// Runs the test and updates the test result.
void Test::Run() {
  internal::HandleExceptionsInMethodIfSupported(this, &Test::SetUp, "SetUp()");
  // We will run the test only if SetUp() was successful.
  if (!HasFatalFailure()) {
    impl->os_stack_trace_getter()->UponLeavingGTest();
    internal::HandleExceptionsInMethodIfSupported(
        this, &Test::TestBody, "the test body");
  }
 
  // However, we want to clean up as much as possible.  Hence we will
  // always call TearDown(), even if SetUp() or the test body has
  // failed.
  impl->os_stack_trace_getter()->UponLeavingGTest();                       internal::HandleExceptionsInMethodIfSupported(
      this, &Test::TearDown, "TearDown()");
}

```
可以看到在Test::Run里面，首先调用的是SetUp，然后执行TestBody，最后执行TearDown。

## 总结

gtest功能强大，代码也很清晰简介。虽然只看了冰山一角，但是也差不多能明白google test framwork的工作原理了。收货满满
