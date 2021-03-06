CppUTest支持与mock一起构建。这篇文档介绍CppUTest对于mock的支持。如下是mock支持的设计目标：

- 与CppUTest相同的设计目的——少量C++特性以便更好的支持嵌入式软件。
- 没有代码生成。
- 没有，或者极少隐藏魔术宏
- 易于使用
- 便于开发

最为主要设计原则是让手动mock更容易，而不是为了自动mock。如果手动mock更容易，使mock自动化是迟早的事，但那本身并不是目的。

### 内容列表
- [简单场景]()
- [使用对象]()
- [使用参数]()
- [将对象作为参数]()
- [输出参数]()
- [返回值]()
- [传递其他数据]()
- [其他的MockSupport]()
- [MockSupport范围]()
- [Mock插件]()
- [C接口]()

### 一个简单场景
一个最简单的场景是检查一个预期的函数是否被调用，如下：

```
#include "CppUTest/TestHarness.h"
#include "CppUTestExt/MockSupport.h"

TEST_GROUP(MockDocumentation)
{
    void teardown()
    {
        mock().clear();
    }
};

void productionCode()
{
    mock().actualCall("productionCode");
}

TEST(MockDocumentation, SimpleScenario)
{
    mock().expectOneCall("productionCode");
    productionCode();
    mock().checkExpectations();
}

```

使用mock时你需要做的仅仅是将头文件CppUTestExt/MockSupport.h包含到测试文件当中，该头文件几乎包括了所有CppUTest Mocking需要的内容。而TEST_GROUP的声明与之前保持一致，并不需要任何改变。

TEST（MockDocumentation, SimpleScenario)中包含了如下的期望语句：

```
mock().expectOneCall("productionCode");
```

在调用mock()时会返回被用来记录期望语句的全局MockSupport（后面会较详细的介绍）。在上面这个例子当中，调用了expectOneCall("productionCode")，它记录了一个“仅仅调用一次productionCode函数”的期望。

函数productionCode的调用与被测试代码对应（译注：用来判断被测试代码中是否真的有对应的函数调用）。该测试用例以检测“是否期望被满足”作为结束：

```
mock().checkExpectations();
```

如果在当前的测试工程没有使用MockSupportPlugin，那么你得在每一个测试用例当中均添加上面这一行用来检测期望是否被满足的代码，反之则不需要。同样的，只要你使用了MockSupport插件，对于在teardown中调用的mock().clear()也可以省略。当你即没有使用MockSupportPlugin，也没有添加mock().clear()语句时，内存泄漏检测器就会将mock调用识别为内存泄漏。

一个被mock过的函数看起来像这样：

```
void productionCode()
{
    mock().actualCall("productionCode");
}
```

上面的代码使用MockSupport来记录针对productionCode函数的实际调用（译注：前面提到过，我们是通过调用mock()来使用MockSupport这一全局变量的）。

这种简单的场景经常被使用到，尤其在给项目中用到的第三方库打桩的时候。

如果针对函数productionCode的调用并没有发生，那么该单元测试会失败并且提示如下的错误：

```
ApplicationLib/MockDocumentationTest.cpp:41: error: Failure in TEST(MockDocumentation, SimpleScenario)
    Mock Failure: Expected call did not happen.
    EXPECTED calls that did NOT happen:
        productionCode -> no parameters
    ACTUAL calls that did happen:
        <none>
```

### 使用对象
在mock对象时，它与mock函数并没有什么不同。下面的代码展示了一个使用运行时mock的例子：

```
class ClassFromProductionCodeMock : public ClassFromProductionCode
{
public:
    virtual void importantFunction()
    {
        mock().actualCall("importantFunction");
    }
};

TEST(MockDocumentation, SimpleScenarioObject)
{
    mock().expectOneCall("importantFunction");

    ClassFromProductionCode* object = new ClassFromProductionCodeMock; /* create mock instead of real thing */
    object->importantFunction();
    mock().checkExpectations();

    delete object;
}
```

该代码示例非常清楚明白。真正的对象被手动创建的一个mock对象替代了，并且在替代对象中通过MockSupport记录了真实的调用。

在使用对象的时候，我们可以通过如下的语句来检查函数调用是否发生在特定的对象上面：

```
mock().expectOneCall("importantFunction").onObject(object);
```

同时，在mock调用的地方需要添加如下的改动：

```
mock().actualCall("importantFunction").onObject(this);
```

当一个函数调用并没有发生在它所期望的对象身上时，就会有如下的错误信息出现：

```
MockFailure: Function called on a unexpected object: importantFunction
    Actual object for call has address: <0x1001003e8>
    EXPECTED calls that DID NOT happen related to function: importantFunction
        (object address: 0x1001003e0)::importantFunction -> no parameters
    ACTUAL calls that DID happen related to function: importantFunction
        <none>
```

### 参数
显然，仅仅去检测一个函数是否被调用看起来还是缺乏一定吸引力的，不过像下面这样可以记录一个函数的参数就很有用了：

```
mock().expectOneCall("function").onObject(object).withParameter("p1", 2).withParameter("p2", "hah");
```

对应mock调用的地方应该写成下面这样：

```
mock().actualCall("function").onObject(this).withParameter("p1", p1).withParameter("p2", p2);
```

在执行测试用例的时候，如果其中某个参数检测不通过，会有如下的错误提示：

```
Mock Failure: Expected parameter for function "function" did not happen.
    EXPECTED calls that DID NOT happen related to function: function
        (object address: 0x1)::function -> int p1: <2>, char* p2: <hah>
    ACTUAL calls that DID happen related to function: function
        <none>
    MISSING parameters that didn't happen:
        int p1, char* p2
```

### 将对象做为参数
withParameters有一个不足：它仅仅支持`int`, `double`, `const char *`或者`void*`。然而，除了这些基本类型，很多时候参数会是一些自定义类型的对象。这个时候该如何处理？下面是一个例子：

```
mock().expectOneCall("function").withParameterOfType("myType", "parameterName", object);
```

可见，这个时候会使用到另外一个接口:withParameterOfType。由于mock框架需要知道如何去比较对应的类型，在操作该类型的参数前必须安装好对应的比较器。

```
MyTypeComparator comparator;
mock().installComparator("myType", comparator);
```

其中的MyTypeComparator是一个自定义的比较器，它需要实现MockNamedValueComparator接口，例如：

```
class MyTypeComparator : public MockNamedValueComparator
{
public:
    virtual bool isEqual(void* object1, void* object2)
    {
        return object1 == object2;
    }
    virtual SimpleString valueToString(void* object)
    {
        return StringFrom(object);
    }
};
```

其中的isEqual用来比较两个参数。而valueToString会在用例执行失败的时候用来打印错误信息，也就是搞定如何输出它期望的值与实际的值。如果你更情愿使用C函数，只需要使用接受构造器中函数指针的MockFunctionComparator。
<i style="color:red">怎么理解这句话，源代码中是怎样的呢？是否在框架里面定义了对应的比较器的指针，只需要将比较器作为参数传递进入就好了？</i>

在需要删除比较器的时候，仅仅需要添加如下一行代码即可：

```
mock().removeAllComparators();
```

比较器有时会让人摸不着头脑，这里有一些使用提示：

- <i>提示一：注意比较器变量的作用域！</i>
比较器在传递参数的时候并不会进行值拷贝，而是直接使用从installComparator函数中传递进来的实例。因此，在使用的时候务必确保实例的有效性。比如，如果你在TEST当中调用了installComparator，却在teardown的时候进行checkExpectations，此时比较器已经销毁了，所以这个时候就会使得测试程序崩溃了。

- <i>提示二：在使用MockPlugin的时候也别忽视其作用域</i>
在使用MockPlugin时（推荐），一种可取的方式是通过MockPlugin来安装比较器，或者将比较器置于全局空间下。checkExpectation会在teardown之后调用，如果比较器在之前销毁也将造成崩溃。

### 出参
对于某些参数，它们传递给调用函数时不是直接传递参数对应的值，而是传递引用，如此便可通过修改它所指向的数据来达到“返回值”的效果。

CppUMock支持在expected call时指定这些输出参数：

```
int outputValue = 4;
mock().expectOneCall("Foo").withOutputParameterReturning("bar", &outputValue, sizeof(outputValue));
```

同时，在actual call当中需要有下面这样对应的代码：

```
void Foo(int *bar)
{
    mock().actualCall("foo").withOutputParameter("bar", bar);
}
```

一旦在调用了actual call之后之后，传递给Foo函数的参数bar的值将会与expected call当中指定的值一致（在这个例子当中bar的值将为4）。

- <i>提示一：</i>
在使用输出参数的时候，CppUMock对于非法访问内存是束手无策的，它会对在withOutputParameterReturning调用中指定大小的内存执行memcpy操作。因此，如果它拷贝的内存大小超过了actual call中出参指向的数据空间大小，这个时候便极有可能会出现段错误了。

针对withOutputParameterReturning提供了针对char, int, unsigned, long, unsigned long和double类型的支持，这样就可以省略掉size这个参数。（译注：这也在一定程度上避免了不小心指定不合理的空间大小而导致段错误的问题）

```
char charOutputValue = 'a';
mock().expectOneCall("Foo").withOutputParameterReturning("bar", &charOutputValue);

int intOutputValue = 4;
mock().expectOneCall("Foo").withOutputParameterReturning("bar", &intOutputValue);

unsigned unsignedOutputValue = 4;
mock().expectOneCall("Foo").withOutputParameterReturning("bar", &unsignedOutputValue);

long longOutputValue = 4;
mock().expectOneCall("Foo").withOutputParameterReturning("bar", &longOutputValue);

unsigned long unsignedLongOutputValue = 4;
mock().expectOneCall("Foo").withOutputParameterReturning("bar", &unsignedLongOutputValue);

double doubleOutputValue = 4;
mock().expectOneCall("Foo").withOutputParameterReturning("bar", &doubleOutputValue);
```

- <i>提示二：</i>
当有char, int等等数组传递给withOutputParameter的时候，你必须在withOutputParameterReturning的时候指定该数组的确切空间，否则它只会拷贝一个元素。

### 返回值
有时需要让一个被mock的函数返回特定的值以便更全的测试到产品代码中的流程。此时测试代码需要如此写：

```
mock().expectOneCall("function").andReturnValue(10);
```

同时被mock的函数需要做如下更新：

```
int function () {
    return mock().actualCall("function").returnIntValue();
}
```

我们也可以像下面这样将returnIntValue和actualCall分开：

```
int function () {
    mock().actualCall("function");
    return mock().intReturnValue();
}
```

CppUMock框架支持的返回值选项用来在测试用例与被mock的对象之间传递数据，它们本身不存在使得测试用例失败的情况。

### 传递其他数据
当需要给一个mock对象传递更多数据的时候，比如在某次计算当中仅仅需要改变一对参数的值，可以做如下处理：

```
ClassFromProductionCode object;
mock().setData("importantValue", 10);
mock().setDataObject("importantObject", "ClassFromProductionCode", &object);
```

在mock对象当中做对应的修改：

```
ClassFromProductionCode * pobject;
int value = mock().getData("importantValue").getIntValue();
pobject = (ClassFromProductionCode*) mock().getData("importantObject").getObjectPointer();
```

与返回值一样，如上这样设置特定的值或者对象并不会导致用例失败，相反却可以给构建mock对象时提供诸多支持。

### 其他MockSupport - ignoring, enabling, clearing, crashing
MockSupport额外提供了不少有用的函数，这一节将一一介绍。

最为常见的，你仅仅需要检测当前测试用例当中的一些调用而不用理会除此之外的其他函数的调用情况。毕竟，如果你为每一个调用都添加一条expectOneCall语句，你的测试用例将非常庞大（事实上让人忧虑的是测试用例过大而导致的代码坏味道）。此时，一种方式便是使用ignoreOtherCalls：

```
mock().expectOneCall("foo");
mock().ignoreOtherCalls();
```


这样做就会使得在执行调用检查的时候仅仅检测函数foo是否得以调用，而其他函数调用的检测将被忽略（比如函数bar）。

有时，你想暂时性的去使能mock框架以便有时间干点其他事情，而不是在整个测试用例执行过程中忽略掉其他调用。这种情况一般发生在初始化阶段，这个时候你可以使用mock的使能/去使能功能：

```
mock().disable();
doSomethingThatWouldOtherwiseBlowUpTheMockingFramework();
mock().enable();
```

在你忽略有返回值的mock函数的调用时，这在测试时可能会引起运行时错误。

这很容易解释，比如你mock了如下的函数：

```
int function () {
    return mock().actualCall("function").returnIntValue();
}
```

如果你试着去忽略它或者去使能mock框架，问题便随之而来。为什么呢？因为此时返回值是未定义的，也就是说这里没有办法去定义被mock函数的返回值。

对于这种情况有不少可以指定默认返回值的函数可供使用，尤其对于这种mock函数被忽略的情况，例如上面的例子可以修改为：

```
int function () {
    return mock().actualCall("function").returnIntValueOrDefault(5);
}
```

在函数被忽略了或者框架被去使能，会返回5的默认值。反之，则返回值为测试用例当中expect指定的值。

除了上面这种方法之外，还可以在mock时直接指定返回值（不需要使用调用actualCall返回的对象）。

```
int function () {
    mock().actualCall("function");
    return mock().returnIntValueOrDefault(5);
}
```

在你需要清除所有的expectations, settings,以及comparators，调用如下clear接口即可：

```
mock().clear();
```

该清楚调用并不会对expectations做任何检测工作，它仅仅会擦除之前的所有mock设置并且重置它们。一般来说clear会在检测了expectations之后被调用。

在一个actual调用产生时你并不会知道它是在什么地方被调用，除了你有拥有该次调用的调用栈。但实际上，mock框架并不支持这样的操作。这里唯一可以使用的是调用crashOnFailure，这样就可以使用调试器去查看对应的调用信息了：

```
mock().crashOnFailure();
```

如果你使用gdb，可以如此做：
```
gdb examples/CppUTestExamples_tests
r
bt
```

（r是run的意思，它会执行测试用例且在其崩溃时推出。bt指的是back trace，它便会打印出调用栈信息了。）

### MockSupport作用域
MockSupport作用域这一特性使得分层使用MockSupport成为可能，这可能看起来很复杂，而实际上并不是那样。在mock函数时，你可以传递命名空间或者作用域并在其中做一些比如记录expectations等事情，例如：

```
mock("xmlparser").expectOneCall("open");
```

相应地，actual调用必须对应起来：

```
mock("xmlparser").actualCall("open");
```

这样指定了作用域之后，就不会受到其他的作用域的影响了。比如下面的actualCall调用就不会匹配到xmlparser当中的open调用：

```
mock("").actualCall("open");
```

在不同的命名空间当中进行mock调用使得ignore某种类型的调用变得容易，同时也就可以更好的打理当前需要关注的mock逻辑了，例如：

```
mock("xmlparser").expectOneCall("open");
mock("filesystem").ignoreOtherCalls();
```

### MockPlugin
CppUTest插件用来增添更多“扩展”功能，它们一般被配置在main函数当中，在那里可以使得对应的功能被应用到所有的单元测试例中。MockPlugin便是一款使得mock更易使用的插件，它包括如下功能：

- 在每一个测试用例的最后进行checkExpectations的工作（如果布置在全局作用域，它会递归的遍历所有的作用域）
- 在每一个测试用例执行完之后清除所有的expectations
- 在每个测试用例的开头安装配置在插件当中的所有比较器
- 在每个测试用例执行完之后移除所有的比较器

安装MockPlugin你得在main函数当中添加类似如下代码：

```
#include "CppUTest/TestRegistry.h"
#include "CppUTestExt/MockSupportPlugin.h"

MyDummyComparator dummyComparator;
MockSupportPlugin mockPlugin;

mockPlugin.installComparator("MyDummyType", dummyComparator);
TestRegistry::getCurrentRegistry()->installPlugin(&mockPlugin);
```

如上为MyDummy创建了一个比较器并将其安装在mockPlugin插件当中，这样就使得该比较器对所有的测试用例可见。之后创建了mockPlugin插件并且注册到当前的测试注册表。一旦完成了这些工作，在checkExpectations以及清除MockSupport这些琐碎的事情上就省心了。

### C接口
通常来说mock框架是基于C++之上，但有时你需要从一个C文件当中使用mock。比如，当一些桩文件是在一些C源文件当中实现的时候，去将它们用C++重写一遍显然费力不讨好，这个时候如果可以在C文件当中直接调用mock框架显然容易许多，而C接口便是为此而生。这些接口基于C++已有的接口写成，所以你可以很明显的知道下面这些代码的作用：

```
#include "CppUTestExt/MockSupport_c.h"

mock_c()->expectOneCall("foo")->withIntParameters("integer", 10)->andReturnDoubleValue(1.11);
mock_c()->actualCall("foo")->withIntParameters("integer", 10)->returnValue().value.doubleValue;

mock_c()->installComparator("type", equalMethod, toStringMethod);
mock_scope_c("scope")->expectOneCall("bar")->withParameterOfType("type", "name", object);
mock_scope_c("scope")->actualCall("bar")->withParameterOfType("type", "name", object);
mock_c()->removeAllComparators();

mock_c()->setIntData("important", 10);

mock_c()->checkExpectations();
mock_c()->clear();
```

C接口使用与C++接口类似的编译结构，尽管比较起来在通用性上面略微逊色，但完全具有应该有的功能。




