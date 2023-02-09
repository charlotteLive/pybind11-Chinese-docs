# pybind11使用指南

## 1. 基础用法

### 1.1 安装与编译
在安装python3-dev和下载了pybind11源码的前提下，可以直接include pybind11头文件目录和python3头文件目录即可。cmake示例如下：
```cmake
set(PYTHON_TARGET_VER 3.6)
find_package(PythonInterp ${PYTHON_TARGET_VER} EXACT)
find_package(PythonLibs ${PYTHON_TARGET_VER} EXACT REQUIRED)

include_directories(pybind11_include_path)
include_directories(${PYTHON_INCLUDE_DIRS})
```

### 1.2 绑定函数

```c++
#include <pybind11/pybind11.h>

int add(int i, int j) {
    return i + j;
}

PYBIND11_MODULE(example, m) {
    m.doc() = "pybind11 example plugin"; // optional module docstring
    m.def("add", &add, "A function which adds two numbers");
}
```
宏`PYBIND11_MODULE`会创建模块初始化函数，它在Python中`import`模块时被调用。其参数分别是模块名，类型为`py::module_`的变量（m），是创建绑定的主要接口。`module_::def()`方法，可以生成函数的绑定。

#### 1.2.1 关键字参数

使用`py::arg`可以指定函数的参数名，Python侧调用函数时可以使用关键字参数，以增加代码的可读性。
```c++
m.def("add", &add, "A function which adds two numbers",
      py::arg("i"), py::arg("j"));
```

更简洁的写法：
```c++
// regular notation
m.def("add1", &add, py::arg("i"), py::arg("j"));
// shorthand
using namespace pybind11::literals;
m.def("add2", &add, "i"_a, "j"_a);
```

Python使用示例：
```python
import example
example.add(i=1, j=2)  #3L
```

#### 1.2.2 参数默认值

pybind11不能自动地提取默认参数，因为它不属于函数类型信息的一部分。我们需要借助`arg`在绑定时指定参数默认值：
```c++
m.def("add", &add, "A function which adds two numbers",
      py::arg("i") = 1, py::arg("j") = 2);
```

更简短的声明方式：
```c++
// regular notation
m.def("add1", &add, py::arg("i") = 1, py::arg("j") = 2);
// shorthand
m.def("add2", &add, "i"_a=1, "j"_a=2);
```

#### 1.2.3 重载函数

重载方法即拥有相同的函数名，但入参不一样的函数。


在绑定重载函数时，我们需要增加函数签名相关的信息以消除歧义。绑定多个函数到同一个Python名称，将会自动创建函数重载链。Python将会依次匹配，找到最合适的重载函数。

```c++
m.def("add", static_cast<int(*)(int, int)>(&add), "A function which adds two int numbers");
m.def("add", static_cast<double(*)(double, double)>(&add), "A function which adds two double numbers");
```

如果你的编译器支持C++14，也可以使用下面的语法来转换重载函数：
```c++
m.def("add", py::overload_cast<int, int>(&add), "A function which adds two int numbers");
m.def("add", py::overload_cast<double, double>(&add), "A function which adds two double numbers");
```

这里，`py::overload_cast`仅需指定函数类型，不用给出返回值类型，以避免原语法带来的不必要的干扰(`void (Pet::*)`)。如果是基于const的重载，需要使用`py::const`标识。

### 1.3 导出变量

我们可以使用`attr`函数来注册需要导出到Python模块中的C++变量。内建类型和常规对象会在指定attriutes时自动转换，也可以使用`py::cast`来显式转换。

```c++
PYBIND11_MODULE(example, m) {
    m.attr("the_answer") = 42;
    py::object world = py::cast("World");
    m.attr("what") = world;
}
``

Python中使用如下：
​```pyhton
>>> import example
>>> example.the_answer
42
>>> example.what
'World'
```

### 1.4 绑定类或结构体

现在我们来绑定一个C++自定义数据结构`Pet`。定义如下：

```c++
struct Pet {
    Pet(const std::string &name) : name(name) { }
    void setName(const std::string &name_) { name = name_; }
    const std::string &getName() const { return name; }

    std::string name;
};
```

绑定代码如下所示：

```c++
#include <pybind11/pybind11.h>
namespace py = pybind11;

PYBIND11_MODULE(example, m) {
    py::class_<Pet>(m, "Pet")
        .def(py::init<const std::string &>())
        .def("setName", &Pet::setName)
        .def("getName", &Pet::getName)
        .def("__repr__",
            [](const Pet &a) {
                return "<example.Pet named '" + a.name + "'>";
            });
}
```

`class_`会创建C++ class或 struct的绑定。`init()`方法用于创建绑定类的构造函数，它使用类构造函数的参数类型作为模板参数，并包装相应的构造函数。

使用`print(p)`打印对象信息时，默认会得到一些没用的信息。我们可以绑定一个工具函数到`__repr__`方法，来返回可读性好的摘要信息。在不改变Pet类的基础上，使用一个匿名函数来完成这个功能是一个不错的选择。

Python使用示例如下；

```python
>>> import example
>>> p = example.Pet("Molly")
>>> print(p)
<example.Pet named 'Molly'>
>>> p.getName()
u'Molly'
>>> p.setName("Charly")
>>> p.getName()
u'Charly'
```

静态成员函数需要使用`class_::def_static`来绑定。

#### 1.4.1 成员函数

使用`class_::def_readwrite`方法可以导出公有成员变量，使用`class_::def_readonly`方法则可以导出只读成员。


```c++
py::class_<Pet>(m, "Pet")
    .def(py::init<const std::string &>())
    .def_readwrite("name", &Pet::name)
    // ... remainder ...
```

Python中使用示例如下：
```python
>>> p = example.Pet("Molly")
>>> p.name
u'Molly'
>>> p.name = "Charly"
>>> p.name
u'Charly'
```

假设`Pet::name`是一个私有成员变量，向外提供setter和getters方法。

```c++
class Pet {
public:
    Pet(const std::string &name) : name(name) { }
    void setName(const std::string &name_) { name = name_; }
    const std::string &getName() const { return name; }
private:
    std::string name;
};
```

可以使用`class_::def_property()`(只读成员使用`class_::def_property_readonly()`)来定义并私有成员，并生成相应的setter和geter方法：
```c++
py::class_<Pet>(m, "Pet")
    .def(py::init<const std::string &>())
    .def_property("name", &Pet::getName, &Pet::setName)
    // ... remainder ...
```

只写属性通过将read函数定义为nullptr来实现。

相似的方法`class_::def_readwrite_static()`, `class_::def_readonly_static()` `class_::def_property_static()`, `class_::def_property_readonly_static()`用于绑定静态变量和属性。

#### 1.4.2 动态属性

原生的Pyhton类可以动态地获取新属性：
```python
>>> class Pet:
...    name = "Molly"
...
>>> p = Pet()
>>> p.name = "Charly"  # overwrite existing
>>> p.age = 2  # dynamically add a new attribute
```

默认情况下，从C++导出的类不支持动态属性，其可写属性必须是通过`class_::def_readwrite`或`class_::def_property`定义的。试图设置其他属性将产生错误：
```python
>>> p = example.Pet()
>>> p.name = "Charly"  # OK, attribute defined in C++
>>> p.age = 2  # fail
AttributeError: 'Pet' object has no attribute 'age'
```

要让C++类也支持动态属性，我们需要在`py::class_`的构造函数添加`py::dynamic_attr`标识：
```c++
py::class_<Pet>(m, "Pet", py::dynamic_attr())
    .def(py::init<>())
    .def_readwrite("name", &Pet::name);
```

这样，之前报错的代码就能够正常运行了。

```python
>>> p = example.Pet()
>>> p.name = "Charly"  # OK, overwrite value in C++
>>> p.age = 2  # OK, dynamically add a new attribute
>>> p.__dict__  # just like a native Python class
{'age': 2}
```

需要提醒一下，支持动态属性会带来小小的运行时开销。不仅仅因为增加了额外的`__dict__`属性，还因为处理循环引用时需要花费更多的垃圾收集跟踪花销。但是不必担心这个问题，因为原生Python类也有同样的开销。默认情况下，pybind11导出的类比原生Python类效率更高，使能动态属性也只是让它们处于同等水平而已。

#### 1.4.3 重载方法

重载类的方法同上一节的普通函数重载，这里举个实例仅供参考：

```c++
struct Pet {
    Pet(const std::string &name, int age) : name(name), age(age) { }

    void set(int age_) { age = age_; }
    void set(const std::string &name_) { name = name_; }

    std::string name;
    int age;
};

// method 1
py::class_<Pet>(m, "Pet")
   .def(py::init<const std::string &, int>())
   .def("set", static_cast<void (Pet::*)(int)>(&Pet::set), "Set the pet's age")
   .def("set", static_cast<void (Pet::*)(const std::string &)>(&Pet::set), "Set the pet's name");

// method 2
py::class_<Pet>(m, "Pet")
    .def("set", py::overload_cast<int>(&Pet::set), "Set the pet's age")
    .def("set", py::overload_cast<const std::string &>(&Pet::set), "Set the pet's name");
```

### 1.5 绑定枚举类型

对于C风格的枚举类型，绑定示例如下：
```c++
enum Flags {
    Read = 4,
    Write = 2,
    Execute = 1
};

py::enum_<Flags>(m, "Flags", py::arithmetic())
    .value("Read", Flags::Read)
    .value("Write", Flags::Write)
    .value("Execute", Flags::Execute)
    .export_values();
```

`enum_::export_values()`用来导出枚举项到父作用域，C++11的强枚举类型需要跳过这点。

枚举类型的枚举项会被导出到类`__members__`属性中，`name`属性可以返回枚举值的名称的unicode字符串，`str(enum)`也可以做到，但两者的实现目标不同。

### 1.6 接收`*args`和`**kwargs`参数

Python的函数可以接收任意数量的参数和关键字参数：
```python
def generic(*args, **kwargs):
    ...  # do something with args and kwargs
```

我们也可以通过pybind11来创建这样的函数：
```c++
void generic(py::args args, const py::kwargs& kwargs) {
    /// .. do something with args
    if (kwargs)
        /// .. do something with kwargs
}

/// Binding code
m.def("generic", &generic);
```

`py::args`继承自`py::tuple`，`py::kwargs`继承自`py::dict`。



## 2. 函数绑定进阶

### 2.1 返回值策略

Python和C++在管理内存和对象生命周期管理上存在本质的区别。这导致我们在创建返回no-trivial类型的函数绑定时会出问题。仅通过类型信息，我们无法明确是Python侧需要接管返回值并负责释放资源，还是应该由C++侧来处理。因此，pybind11提供了一些返回值策略来确定由哪方管理资源。这些策略通过`model::def()`和`class_def()`来指定，默认策略为`return_value_policy::automatic`。

返回值策略难以捉摸，正确地选择它们则显得尤为重要。下面我们通过一个简单的例子来阐释选择错误的情形：
```c++
/* Function declaration */
Data *get_data() { return _data; /* (pointer to a static data structure) */ }
...

/* Binding code */
m.def("get_data", &get_data); // <-- KABOOM, will cause crash when called from Python
```

当Python侧调用`get_data()`方法时，返回值（原生C++类型）必须被转换为合适的Python类型。在这个例子中，默认的返回值策略（`return_value_policy::automatic`）使得pybind11获取到了静态变量`_data`的所有权。

当Python垃圾收集器最终删除`_data`的Python封装时，pybind11将尝试删除C++实例（通过operator delete()）。这时，这个程序将以某种隐蔽的错误并涉及静默数据破坏的方式崩溃。

对于上面的例子，我们应该指定返回值策略为`return_value_policy::reference`，这样全局变量的实例仅仅被引用，而不涉及到所有权的转移：
```c++
m.def("get_data", &get_data, py::return_value_policy::reference);
```

另一方面，引用策略在多数其他场合并不是正确的策略，忽略所有权的归属可能导致资源泄漏。作为一个使用pybind11的开发者，熟悉不同的返回值策略及其适用场合尤为重要。下面的表格将提供所有策略的概览：

| 返回值策略                                 | 描述                                                         |
| ------------------------------------------ | ------------------------------------------------------------ |
| `return_value_policy::take_ownership`      | 引用现有对象（不创建一个新对象），并获取所有权。在引用计数为0时，Pyhton将调用析构函数和delete操作销毁对象。 |
| `return_value_policy::copy`                | 拷贝返回值，这样Python将拥有拷贝的对象。该策略相对来说比较安全，因为两个实例的生命周期是分离的。 |
| `return_value_policy::move`                | 使用`std::move`来移动返回值的内容到新实例，新实例的所有权在Python。该策略相对来说比较安全，因为两个实例的生命周期是分离的。 |
| `return_value_policy::reference`           | 引用现有对象，但不拥有所有权。C++侧负责该对象的生命周期管理，并在对象不再被使用时负责析构它。注意：当Python侧还在使用引用的对象时，C++侧删除对象将导致未定义行为。 |
| `return_value_policy::reference_internal`  | 返回值的生命周期与父对象的生命周期相绑定，即被调用函数或属性的`this`或`self`对象。这种策略与reference策略类似，但附加了`keep_alive<0, 1>`调用策略保证返回值还被Python引用时，其父对象就不会被垃圾回收掉。这是由`def_property`、`def_readwrite`创建的属性getter方法的默认返回值策略。 |
| `return_value_policy::automatic`           | 当返回值是指针时，该策略使用`return_value_policy::take_ownership`。反之对左值和右值引用使用`return_value_policy::copy`。请参阅上面的描述，了解所有这些不同的策略的作用。这是`py::class_`封装类型的默认策略。 |
| `return_value_policy::automatic_reference` | 和上面一样，但是当返回值是指针时，使用`return_value_policy::reference`策略。这是在C++代码手动调用Python函数和使用`pybind11/stl.h`中的casters时的默认转换策略。你可能不需要显式地使用该策略。 |

返回值策略也可以应用于属性：
```c++
class_<MyClass>(m, "MyClass")
    .def_property("data", &MyClass::getData, &MyClass::setData,
                  py::return_value_policy::copy);
```

在技术层面，上述代码会将策略同时应用于getter和setter函数，但是setter函数并不关心返回值策略，这样做仅仅出于语法简洁的考虑。或者，你可以通过`cpp_function`构造函数来传递目标参数：
```c++
class_<MyClass>(m, "MyClass")
    .def_property("data"
        py::cpp_function(&MyClass::getData, py::return_value_policy::copy),
        py::cpp_function(&MyClass::setData)
    );
```

**注意**：代码使用无效的返回值策略将导致未初始化内存或多次free数据结构，这将导致难以调试的、不确定的问题和段错误。因此，花点时间来理解上面表格的各个选项是值得的。

**提示**：
1. 上述策略的另一个重点是，他们仅可以应用于pybind11还不知晓的实例，这时策略将澄清返回值的生命周期和所有权问题。当pybind11已经知晓参数（通过其在内存中的类型和地址来识别），它将返回已存在的Python对象封装，而不是创建一份拷贝。
2. 下一节将讨论上面表格之外的调用策略，他涉及到返回值和函数参数的引用关系。
3. 可以考虑使用智能指针来代替复杂的调用策略和生命周期管理逻辑。智能指针会告诉你一个对象是否仍被C++或Python引用，这样就可以消除各种可能引发crash或未定义行为的矛盾。对于返回智能指针的函数，没必要指定返回值策略。

### 2.2 附加的调用策略

除了以上的返回值策略外，进一步指定调用策略可以表明参数间的依赖关系，确保函数调用的稳定性。

#### 保活（keep alive）

当一个C++容器对象包含另一个C++对象时，我们需要使用该策略。`keep_alive<Nurse, Patient>`表明至少在索引Nurse被回收前，索引Patient应该被保活。0表示返回值，1及以上表示参数索引。1表示隐含的参数this指针，而常规参数索引从2开始。当Nurse的值在运行前被检测到为None时，调用策略将什么都不做。

当nurse不是一个pybind11注册类型时，实现依赖于创建对nurse对象弱引用的能力。如果nurse对象不是pybind11注册类型，也不支持弱引用，程序将会抛出异常。

如果你使用一个错误的参数索引，程序将会抛出"Could not cativate keep_alive!"警告的运行时异常。这时，你应该review你代码中使用的索引。

参见下面的例子：一个list append操作，将新添加元素的生命周期绑定到添加的容器对象上：
```c++
py::class_<List>(m, "List").def("append", &List::append, py::keep_alive<1, 2>());
```

为了一致性，构造函数的实参索引也是相同的。索引1仍表示this指针，索引0表示返回值（构造函数的返回值被认为是void）。下面的示例将构造函数入参的生命周期绑定到被构造对象上。
```c++
py::class_<Nurse>(m, "Nurse").def(py::init<Patient &>(), py::keep_alive<1, 2>());
```

> Note: `keep_alive`与Boost.Python中的`with_custodian_and_ward`和`with_custodian_and_ward_postcall`相似。

#### Call guard

`call_guard<T>`策略允许任意T类型的scope guard应用于整个函数调用。示例如下：
```c++
m.def("foo", foo, py::call_guard<T>());
```

上面的代码等价于：
```c++
m.def("foo", [](args...) {
    T scope_guard;
    return foo(args...); // forwarded arguments
});
```

仅要求模板参数T是可构造的，如`gil_scoped_release`就是一个非常有用的类型。

`call_guard`支持同时制定多个模板参数，`call_guard<T1, T2, T3 ...>`。构造顺序是从左至右，析构顺序则相反。

> See also: `test/test_call_policies.cpp`含有更丰富的示例来展示`keep_alive`和`call_guard`的用法。

### 2.3 Keyword-only参数

Python3提供了keyword-only参数（在函数定义中使用`*`作为匿名参数）：
```python
def f(a, *, b):  # a can be positional or via keyword; b must be via keyword
    pass

f(a=1, b=2)  # good
f(b=2, a=1)  # good
f(1, b=2)  # good
f(1, 2)  # TypeError: f() takes 1 positional argument but 2 were given
```

pybind11提供了`py::kw_only`对象来实现相同的功能：
```c++
m.def("f", [](int a, int b) { /* ... */ },
      py::arg("a"), py::kw_only(), py::arg("b"));
```

注意，该特性不能与`py::args`一起使用。

### 2.4 Positional-only参数

python3.8引入了Positional-only参数语法，pybind11通过`py::pos_only()`来提供相同的功能：
```c++
m.def("f", [](int a, int b) { /* ... */ },
       py::arg("a"), py::pos_only(), py::arg("b"));
```

现在，你不能通过关键字来给定`a`参数。该特性可以和keyword-only参数一起使用。

### 2.5 Non-converting参数

有些参数可能支持类型转换，如：
- 通过`py::implicitly_convertible<A,B>()`进行隐式转换
- 将整形变量传给入参为浮点类型的函数
- 将非复数类型（如float）传给入参为`std::complex<float>`类型的函数
- Calling a function taking an Eigen matrix reference with a numpy array of the wrong type or of an incompatible data layout.

有时这种转换并不是我们期望的，我们可能更希望绑定代码抛出错误，而不是转换参数。通过`py::arg`来调用`.noconvert()`方法可以实现这个事情。
```c++
m.def("floats_only", [](double f) { return 0.5 * f; }, py::arg("f").noconvert());
m.def("floats_preferred", [](double f) { return 0.5 * f; }, py::arg("f"));
```

尝试进行转换时，将抛出`TypeError`异常：
```python
>>> floats_preferred(4)
2.0
>>> floats_only(4)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: floats_only(): incompatible function arguments. The following argument types are supported:
    1. (f: float) -> float

Invoked with: 4
```

该方法可以与缩写符号`_a`和默认参数配合使用，像这样`py::arg().noconvert()`。

## 3. 类绑定进阶

### 3.1 继承与多态

现在有两个具有继承关系的类：
```c++
struct Pet {
    Pet(const std::string &name) : name(name) { }
    std::string name;
};

struct Dog : Pet {
    Dog(const std::string &name) : Pet(name) { }
    std::string bark() const { return "woof!"; }
};
```

pybind11提供了两种方法来指明继承关系：1）将C++基类作为派生类`class_`的模板参数；2）将基类名作为`class_`的参数绑定到派生类。两种方法是等效的。

```c++
py::class_<Pet>(m, "Pet")
   .def(py::init<const std::string &>())
   .def_readwrite("name", &Pet::name);

// Method 1: template parameter:
py::class_<Dog, Pet /* <- specify C++ parent type */>(m, "Dog")
    .def(py::init<const std::string &>())
    .def("bark", &Dog::bark);

// Method 2: pass parent class_ object:
py::class_<Dog>(m, "Dog", pet /* <- specify Python parent type */)
    .def(py::init<const std::string &>())
    .def("bark", &Dog::bark);
```

指明继承关系后，派生类实例将获得两者的字段和方法：
```python
>>> p = example.Dog("Molly")
>>> p.name
u'Molly'
>>> p.bark()
u'woof!'
```

上面的例子是一个常规非多态的继承关系，表现在Python就是：
```c++
// 返回一个指向派生类的基类指针
m.def("pet_store", []() { return std::unique_ptr<Pet>(new Dog("Molly")); });
```

```python
>>> p = example.pet_store()
>>> type(p)  # `Dog` instance behind `Pet` pointer
Pet          # no pointer downcasting for regular non-polymorphic types
>>> p.bark()
AttributeError: 'Pet' object has no attribute 'bark'
```

`pet_store`函数返回了一个Dog实例，但由于基类并非多态类型，Python只识别到了Pet。在C++中，一个类至少有一个虚函数才会被视为多态类型。pybind11会自动识别这种多态机制。

```c++
struct PolymorphicPet {
    virtual ~PolymorphicPet() = default;
};

struct PolymorphicDog : PolymorphicPet {
    std::string bark() const { return "woof!"; }
};

// Same binding code
py::class_<PolymorphicPet>(m, "PolymorphicPet");
py::class_<PolymorphicDog, PolymorphicPet>(m, "PolymorphicDog")
    .def(py::init<>())
    .def("bark", &PolymorphicDog::bark);

// Again, return a base pointer to a derived instance
m.def("pet_store2", []() { return std::unique_ptr<PolymorphicPet>(new PolymorphicDog); });
```

```python
>>> p = example.pet_store2()
>>> type(p)
PolymorphicDog  # automatically downcast
>>> p.bark()
u'woof!'
```

pybind11会自动地将一个指向多态基类的指针，向下转型为实际的派生类类型。这和C++常见的情况不同，我们不仅可以访问基类的虚函数，还能获取到通过基类看不到的，具体的派生类的方法和属性。

### 3.2 Python继承C++类

对于一个拥有纯虚函数的类，使用常规的绑定方法并在Python中直接继承它会报错，因为纯虚基类是不可构造的。

```c++
class Animal {
public:
    virtual ~Animal() { }
    virtual std::string go(int n_times) = 0;
};

class Dog : public Animal {
public:
    std::string go(int n_times) override {
        std::string result;
        for (int i=0; i<n_times; ++i)
            result += "woof! ";
        return result;
    }
};

std::string call_go(Animal *animal) {
    return animal->go(3);
}

PYBIND11_MODULE(example, m) {
    py::class_<Animal>(m, "Animal")
        .def("go", &Animal::go);

    py::class_<Dog, Animal>(m, "Dog")
        .def(py::init<>());

    m.def("call_go", &call_go);
}
```

这样绑定之后，用户在Python中继承实现Animal会报错，提示"No constructor defined!"。

这时，我们需要定义一个新的Animal类作为辅助跳板：
```c++
class PyAnimal : public Animal {
public:
    /* Inherit the constructors */
    using Animal::Animal;

    /* Trampoline (need one for each virtual function) */
    std::string go(int n_times) override {
        PYBIND11_OVERRIDE_PURE(
            std::string, /* Return type */
            Animal,      /* Parent class */
            go,          /* Name of function in C++ (must match Python name) */
            n_times      /* Argument(s) */
        );
    }
};
```

定义纯虚函数时需要使用`PYBIND11_OVERRIDE_PURE`宏，而有默认实现的虚函数则使用`PYBIND11_OVERRIDE`。`PYBIND11_OVERRIDE_PURE_NAME` 和`PYBIND11_OVERRIDE_NAME` 宏的功能类似，主要用于C函数名和Python函数名不一致的时候。以`__str__`为例：

```c++
std::string toString() override {
  PYBIND11_OVERRIDE_NAME(
      std::string, // Return type (ret_type)
      Animal,      // Parent class (cname)
      "__str__",   // Name of method in Python (name)
      toString,    // Name of function in C++ (fn)
  );
}
```

Animal类的绑定代码也需要一些微调：

```c++
PYBIND11_MODULE(example, m) {
    py::class_<Animal, PyAnimal /* <--- trampoline*/>(m, "Animal")
        .def(py::init<>())
        .def("go", &Animal::go);

    py::class_<Dog, Animal>(m, "Dog")
        .def(py::init<>());

    m.def("call_go", &call_go);
}
```

pybind11通过向`class_`指定额外的模板参数PyAnimal，让我们可以在Python中继承Animal类。

接下来，我们可以像往常一样定义构造函数。绑定时我们需要使用真实类，而不是辅助类。

```c++
py::class_<Animal, PyAnimal /* <--- trampoline*/>(m, "Animal");
    .def(py::init<>())
    .def("go", &PyAnimal::go); /* <--- THIS IS WRONG, use &Animal::go */
```

下面的Python代码展示了我们继承并重载了`Animal::go`方法，并通过虚函数来调用它：

```python
from example import *
d = Dog()
call_go(d)     # u'woof! woof! woof! '
class Cat(Animal):
    def go(self, n_times):
        return "meow! " * n_times

c = Cat()
call_go(c)   # u'meow! meow! meow! '
```

如果你在派生的Python类中自定义了一个构造函数，你必须保证显示调用C++构造函数(通过`__init__`)，不管它是否为默认构造函数。否则，实例属于C++那部分的内存就未初始化，可能导致未定义行为。在pybind11 2.6版本中，这种错误将会抛出`TypeError`异常。

```python
class Dachshund(Dog):
    def __init__(self, name):
        Dog.__init__(self)  # Without this, a TypeError is raised.
        self.name = name

    def bark(self):
        return "yap!"
```

注意必须显式地调用`__init__`，而不应该使用`supper()`。在一些简单的线性继承中，`supper()`或许可以正常工作；一旦你混合Python和C++类使用多重继承，由于Python MRO和C++的机制，一切都将崩溃。

### 3.3 虚函数与继承

综合考虑虚函数与继承时，你需要为每个你允许在Python派生类中重载的方法提供重载方式。下面我们扩展Animal和Dog来举例：

```c++
class Animal {
public:
    virtual std::string go(int n_times) = 0;
    virtual std::string name() { return "unknown"; }
};
class Dog : public Animal {
public:
    std::string go(int n_times) override {
        std::string result;
        for (int i=0; i<n_times; ++i)
            result += bark() + " ";
        return result;
    }
    virtual std::string bark() { return "woof!"; }
};
```

上节涉及到的Animal辅助类仍是必须的，为了让Python代码能够继承`Dog`类，我们也需要为`Dog`类增加一个跳板类，来实现`bark()`和继承自Animal的`go()`、`name()`等重载方法（即便Dog类并不直接重载name方法）。

```c++
class PyAnimal : public Animal {
public:
    using Animal::Animal; // Inherit constructors
    std::string go(int n_times) override { PYBIND11_OVERRIDE_PURE(std::string, Animal, go, n_times); }
    std::string name() override { PYBIND11_OVERRIDE(std::string, Animal, name, ); }
};
class PyDog : public Dog {
public:
    using Dog::Dog; // Inherit constructors
    std::string go(int n_times) override { PYBIND11_OVERRIDE(std::string, Dog, go, n_times); }
    std::string name() override { PYBIND11_OVERRIDE(std::string, Dog, name, ); }
    std::string bark() override { PYBIND11_OVERRIDE(std::string, Dog, bark, ); }
};
```

> 注意到`name()`和`bark()`尾部的逗号，这用来说明辅助类的函数不带任何参数。当函数至少有一个参数时，应该省略尾部的逗号。

注册一个继承已经在pybind11中注册的带虚函数的类，同样需要为其添加辅助类，即便它没有定义或重载任何虚函数：

```c++
class Husky : public Dog {};
class PyHusky : public Husky {
public:
    using Husky::Husky; // Inherit constructors
    std::string go(int n_times) override { PYBIND11_OVERRIDE_PURE(std::string, Husky, go, n_times); }
    std::string name() override { PYBIND11_OVERRIDE(std::string, Husky, name, ); }
    std::string bark() override { PYBIND11_OVERRIDE(std::string, Husky, bark, ); }
};
```

我们可以使用模板辅助类将简化这类重复的绑定工作，这对有多个虚函数的基类尤其有用：

```c++
template <class AnimalBase = Animal> class PyAnimal : public AnimalBase {
public:
    using AnimalBase::AnimalBase; // Inherit constructors
    std::string go(int n_times) override { PYBIND11_OVERRIDE_PURE(std::string, AnimalBase, go, n_times); }
    std::string name() override { PYBIND11_OVERRIDE(std::string, AnimalBase, name, ); }
};
template <class DogBase = Dog> class PyDog : public PyAnimal<DogBase> {
public:
    using PyAnimal<DogBase>::PyAnimal; // Inherit constructors
    // Override PyAnimal's pure virtual go() with a non-pure one:
    std::string go(int n_times) override { PYBIND11_OVERRIDE(std::string, DogBase, go, n_times); }
    std::string bark() override { PYBIND11_OVERRIDE(std::string, DogBase, bark, ); }
};
```

这样，我们只需要一个辅助方法来定义虚函数和纯虚函数的重载了。只是这样编译器就需要生成许多额外的方法和类。

下面我们在pybind11中注册这些类：

```c++
py::class_<Animal, PyAnimal<>> animal(m, "Animal");
py::class_<Dog, Animal, PyDog<>> dog(m, "Dog");
py::class_<Husky, Dog, PyDog<Husky>> husky(m, "Husky");
// ... add animal, dog, husky definitions
```

注意，Husky不需要一个专门的辅助类，因为它没定义任何新的虚函数和纯虚函数的重载。

Python中的使用示例：

```python
class ShihTzu(Dog):
    def bark(self):
        return "yip!"
```

### 3.4 非公有析构函数

如果一个类拥有私有或保护的析构函数（例如单例类），通过pybind11绑定类时编译器将会报错。本质的问题是`std::unique_ptr`智能指针负责管理实例的生命周期需要引用析构函数，即便没有资源需要回收。Pybind11提供了辅助类`py::nodelete`来禁止对析构函数的调用。这种情况下，C++侧负责析构对象避免内存泄漏就十分重要。

```c++
/* ... definition ... */

class MyClass {
private:
    ~MyClass() { }
};

/* ... binding code ... */

py::class_<MyClass, std::unique_ptr<MyClass, py::nodelete>>(m, "MyClass")
    .def(py::init<>())
```

### 3.5 隐式转换

假设项目中有A和B两个类型，A可以直接转换为B。

```c++
py::class_<A>(m, "A")
    /// ... members ...

py::class_<B>(m, "B")
    .def(py::init<A>())
    /// ... members ...

m.def("func",
    [](const B &) { /* .... */ }
);
```

如果想func函数传入A类型的参数a，Pyhton侧需要这样写`func(B(a))`，而C++则可以直接使用`func(a)`，自动将A类型转换为B类型。

这种情形下（B有一个接受A类型参数的构造函数），我们可以使用如下声明来让Python侧也支持类似的隐式转换：

```c++
py::implicitly_convertible<A, B>();
```

### 3.6 重载操作符

假设有这样一个类`Vector2`，它通过重载操作符实现了向量加法和标量乘法。

```c++
class Vector2 {
public:
    Vector2(float x, float y) : x(x), y(y) { }

    Vector2 operator+(const Vector2 &v) const { return Vector2(x + v.x, y + v.y); }
    Vector2 operator*(float value) const { return Vector2(x * value, y * value); }
    Vector2& operator+=(const Vector2 &v) { x += v.x; y += v.y; return *this; }
    Vector2& operator*=(float v) { x *= v; y *= v; return *this; }

    friend Vector2 operator*(float f, const Vector2 &v) {
        return Vector2(f * v.x, f * v.y);
    }

    std::string toString() const {
        return "[" + std::to_string(x) + ", " + std::to_string(y) + "]";
    }
private:
    float x, y;
};
```

操作符绑定代码如下：

```python
#include <pybind11/operators.h>

PYBIND11_MODULE(example, m) {
    py::class_<Vector2>(m, "Vector2")
        .def(py::init<float, float>())
        .def(py::self + py::self)
        .def(py::self += py::self)
        .def(py::self *= float())
        .def(float() * py::self)
        .def(py::self * float())
        .def(-py::self)
        .def("__repr__", &Vector2::toString);
}
```

`.def(py::self * float())`是如下代码的简短标记：

```c++
.def("__mul__", [](const Vector2 &a, float b) {
    return a * b;
}, py::is_operator())
```

### 3.7 深拷贝支持

Python通常在赋值中使用引用。有时需要一个真正的拷贝，以防止修改所有的拷贝实例。`copy`模块提供了这样的拷贝能力。

在Python3中，带pickle支持的类自带深拷贝能力。但是，自定义`__copy__`和`__deepcopy__`方法能够提高拷贝的性能。在Python2.7中，由于pybind11只支持cPickle，要想实现深拷贝，自定义这两个方法必须实现。

对于一些简单的类，可以使用拷贝构造函数来实现深拷贝。如下所示：

```c++
py::class_<Copyable>(m, "Copyable")
    .def("__copy__",  [](const Copyable &self) {
        return Copyable(self);
    })
    .def("__deepcopy__", [](const Copyable &self, py::dict) {
        return Copyable(self);
    }, "memo"_a);
```

> Note: 本例中不会复制动态属性。

### 3.8 多重继承

pybind11支持绑定多重继承的类，只需在将所有基类作为`class_`的模板参数即可：

```c++
py::class_<MyType, BaseType1, BaseType2, BaseType3>(m, "MyType")
   ...
```

基类间的顺序任意，甚至可以穿插使用别名或者holder类型，pybind11能够自动识别它们。唯一的要求就是第一个模板参数必须是类型本身。

允许Python中定义的类继承多个C++类，也允许混合继承C++类和Python类。

有一个关于该特性实现的警告：当仅指定一个基类，实际上有多个基类时，pybind11会认为它并没有使用多重继承，这将导致未定义行为。对于这个问题，我们可以在类构造函数中添加`multiple_inheritance`的标识。

```c++
py::class_<MyType, BaseType2>(m, "MyType", py::multiple_inheritance());
```

当模板参数列出了多个基类时，无需使用该标识。

### 3.9 绑定protected成员函数

通常不可能向Python公开protected 成员函数：

```c++
class A {
protected:
    int foo() const { return 42; }
};

py::class_<A>(m, "A")
    .def("foo", &A::foo); // error: 'foo' is a protected member of 'A'
```

因为非公有成员函数意味着外部不可调用。但我们还是希望在Python派生类中使用protected 函数。我们可以通过下面的方式来实现：

```c++
class A {
protected:
    int foo() const { return 42; }
};

class Publicist : public A { // helper type for exposing protected functions
public:
    using A::foo; // inherited with different access modifier
};

py::class_<A>(m, "A") // bind the primary class
    .def("foo", &Publicist::foo); // expose protected methods via the publicist
```

因为 `&Publicist::foo` 和`&A::foo` 准确地说是同一个函数（相同的签名和地址），仅仅是获取方式不同。 `Publicist` 的唯一意图，就是将函数的作用域变为`public`。

如果是希望公开在Python侧重载的 `protected`虚函数，可以将publicist pattern与之前提到的trampoline相结合：

```c++
class A {
public:
    virtual ~A() = default;

protected:
    virtual int foo() const { return 42; }
};

class Trampoline : public A {
public:
    int foo() const override { PYBIND11_OVERRIDE(int, A, foo, ); }
};

class Publicist : public A {
public:
    using A::foo;
};

py::class_<A, Trampoline>(m, "A") // <-- `Trampoline` here
    .def("foo", &Publicist::foo); // <-- `Publicist` here, not `Trampoline`!
```

### 3.10 绑定final类

在C++11中，我们可以使用`findal`关键字来确保一个类不被继承。`py::is_final`属性则可以用来确保一个类在Python中不被继承。底层的C++类型不需要定义为final。

```c++
class IsFinal final {};

py::class_<IsFinal>(m, "IsFinal", py::is_final());
```

在Python中试图继承这个类，将导致错误：

```python
class PyFinalChild(IsFinal):
    pass

TypeError: type 'IsFinal' is not an acceptable base type
```

## 4. 异常处理

### 4.1 C++内置异常到Python异常的转换

当Python通过pybind11调用C++代码时，pybind11将捕获C++异常，并将其翻译为对应的Python异常后抛出。这样Python代码就能够处理它们。

pybind11定义了`std::exception`及其标准子类，和一些特殊异常到Python异常的翻译。由于它们不是真正的Python异常，所以不能使用Python C API来检查。相反，它们是纯C++异常，当它们到达异常处理器时，pybind11将其翻译为对应的Python异常。

| Exception thrown by C++     | Translated to Python exception type                          |
| --------------------------- | ------------------------------------------------------------ |
| `std::exception`            | `RuntimeError`                                               |
| `std::bad_alloc`            | `MemoryError`                                                |
| `std::domain_error`         | `ValueError`                                                 |
| `std::invalid_argument`     | `ValueError`                                                 |
| `std::length_error`         | `ValueError`                                                 |
| `std::out_of_range`         | `IndexError`                                                 |
| `std::range_error`          | `ValueError`                                                 |
| `std::overflow_error`       | `OverflowError`                                              |
| `pybind11::stop_iteration`  | `StopIteration` (used to implement custom iterators)         |
| `pybind11::index_error`     | `IndexError` (used to indicate out of bounds access in `__getitem__`, `__setitem__`, etc.) |
| `pybind11::key_error`       | `KeyError` (used to indicate out of bounds access in `__getitem__`, `__setitem__` in dict-like objects, etc.) |
| `pybind11::value_error`     | `ValueError` (used to indicate wrong value passed in `container.remove(...)`) |
| `pybind11::type_error`      | `TypeError`                                                  |
| `pybind11::buffer_error`    | `BufferError`                                                |
| `pybind11::import_error`    | `ImportError`                                                |
| `pybind11::attribute_error` | `AttributeError`                                             |
| Any other exception         | `RuntimeError`                                               |

异常翻译不是双向的。即上述异常不会捕获源自Python的异常。Python的异常，需要捕获`pybind11::error_already_set`。

这里有个特殊的异常，当入参不能转化为Python对象时，`handle::call()`将抛出`cast_error`异常。

### 4.2 注册定制异常翻译

如果上述默认异常转换策略不够用，pybind11也提供了注册自定义异常翻译的支持。类似于pybind11 class，异常翻译也可以定义在模块内或global。要注册一个使用C++异常的`what()`方法将C++到Python的异常转换，可以使用下面的方法：

```c++
py::register_exception<CppExp>(module, "PyExp");
```

这个调用在指定模块创建了一个名称为PyExp的Python异常，并自动将CppExp相关的异常转换为PyExp异常。

相似的函数可以注册模块内的异常翻译：

```c++
py::register_local_exception<CppExp>(module, "PyExp");
```

方法的第三个参数handle可以指定异常的基类：

```c++
py::register_exception<CppExp>(module, "PyExp", PyExc_RuntimeError);
py::register_local_exception<CppExp>(module, "PyExp", PyExc_RuntimeError);
```

这样，PyExp异常可以捕获PyExp和RuntimeError。

Python内置的异常类型可以参考Python文档[Standard Exceptions](https://docs.python.org/3/c-api/exceptions.html#standard-exceptions)，默认的基类为`PyExc_Exception`。

`py::register_exception_translator(translator)` 和`py::register_local_exception_translator(translator)` 提供了更高级的异常翻译功能，它可以注册任意的异常类型。函数接受一个无状态的回调函数`void(std::exception_ptr)`。

### 9.3 在C++中处理Python异常

当C++调用Python函数时（回调函数或者操作Python对象），若Python有异常抛出，pybind11会将Python异常转化为`pybind11::error_already_set`类型的异常，它包含了一个C++字符串描述和实际的Python异常。`error_already_set`用于将Python异常传回Python（或者在C++侧处理）。

| Exception raised in Python | Thrown as C++ exception type  |
| -------------------------- | ----------------------------- |
| Any Python `Exception`     | `pybind11::error_already_set` |

举个例子：

```c++
try {
    // open("missing.txt", "r")
    auto file = py::module_::import("io").attr("open")("missing.txt", "r");
    auto text = file.attr("read")();
    file.attr("close")();
} catch (py::error_already_set &e) {
    if (e.matches(PyExc_FileNotFoundError)) {
        py::print("missing.txt not found");
    } else if (e.matches(PyExc_PermissionError)) {
        py::print("missing.txt found but not accessible");
    } else {
        throw;
    }
}
```

该方法并不适用与C++到Python的翻译，Python侧抛出的异常总是被翻译为`error_already_set`.

```c++
try {
    py::eval("raise ValueError('The Ring')");
} catch (py::value_error &boromir) {
    // Boromir never gets the ring
    assert(false);
} catch (py::error_already_set &frodo) {
    // Frodo gets the ring
    py::print("I will take the ring");
}

try {
    // py::value_error is a request for pybind11 to raise a Python exception
    throw py::value_error("The ball");
} catch (py::error_already_set &cat) {
    // cat won't catch the ball since
    // py::value_error is not a Python exception
    assert(false);
} catch (py::value_error &dog) {
    // dog will catch the ball
    py::print("Run Spot run");
    throw;  // Throw it again (pybind11 will raise ValueError)
}
```

### 9.4 处理Python C API的错误

尽可能地使用pybind11 wrappers代替直接调用Python C API。如果确实需要直接使用Python C API，除了需要手动管理引用计数外，还必须遵守pybind11的错误处理协议。

在调用Python C API后，如果Python返回错误，需要调用`throw py::error_already_set();`语句，让pybind11来处理异常并传递给Python解释器。这包括对错误设置函数的调用，如`PyErr_SetString`。

```c++
PyErr_SetString(PyExc_TypeError, "C API type error demo");
throw py::error_already_set();

// But it would be easier to simply...
throw py::type_error("pybind11 wrapper type error");
```

也可以调用`PyErr_Clear`来忽略错误。

任何Python错误必须被抛出或清除，否则Python/pybind11将处于无效的状态。

### 9.5 处理unraiseable异常

如果Python调用的C++析构函数或任何标记为`noexcept(true)`的函数抛出了异常，该异常不会传播出去。如果它们在调用图中抛出或捕捉不到任何异常，c++运行时将调用std::terminate()立即终止程序。在C++析构函数中调用Python尤其需要注意异常的捕获，必须捕获所有`error_already_set`类型的异常，并使用`error_already_set::discard_as_unraisable()`来抛弃Python异常。

类似的，在类`__del__`方法引发的Python异常也不会传播，但被Python作为unraisable错误记录下来。在Python 3.8+中，将触发system hook，并记录auditing event日志。

任何noexcept函数应该使用try-catch代码块来捕获`error_already_set`（或其他可能出现的异常）。pybind11包装的Python异常并非真正的Python异常，它是pybind11捕获并转化的C++异常。noexcept函数不能传播这些异常。我们可以将它们转换为Python异常，然后丢弃`discard_as_unraisable`，如下所示。

```c++
void nonthrowing_func() noexcept(true) {
    try {
        // ...
    } catch (py::error_already_set &eas) {
        // Discard the Python error using Python APIs, using the C++ magic
        // variable __func__. Python already knows the type and value and of the
        // exception object.
        eas.discard_as_unraisable(__func__);
    } catch (const std::exception &e) {
        // Log and discard C++ exceptions.
        third_party::log(e);
    }
}
```


## 5. 类型转换


## 6. python C++接口


## 7. 杂项

### 7.1 关于便利宏的说明

pybind11提供了一些便利宏如`PYBIND11_DECLARE_HOLDER_TYPE()`和`PYBIND11_OVERRIDE_*`。由于这些宏只是在预处理中计算(预处理程序没有类型的概念)，它们会被模板参数中的逗号搞混。如：

```c++
PYBIND11_OVERRIDE(MyReturnType<T1, T2>, Class<T3, T4>, func)
```

预处理器会将其解释为5个参数（逗号分隔），而不是3个。有两种方法可以处理这个问题：使用类型别名，或者使用`PYBIND11_TYPE`包裹类型。

```c++
// Version 1: using a type alias
using ReturnType = MyReturnType<T1, T2>;
using ClassType = Class<T3, T4>;
PYBIND11_OVERRIDE(ReturnType, ClassType, func);

// Version 2: using the PYBIND11_TYPE macro:
PYBIND11_OVERRIDE(PYBIND11_TYPE(MyReturnType<T1, T2>),
                  PYBIND11_TYPE(Class<T3, T4>), func)
```

`PYBIND11_MAKE_OPAQUE`宏不需要上述解决方案。

### 7.2 全局解释器锁（GIL）

在Python中调用C++函数时，默认会持有GIL。`gil_scoped_release`和`gil_scoped_acquire`可以方便地在函数体中释放和获取GIL。这样长时间运行的C++代码可以通过Python线程实现并行化。示例如下：

```c++
class PyAnimal : public Animal {
public:
    /* Inherit the constructors */
    using Animal::Animal;

    /* Trampoline (need one for each virtual function) */
    std::string go(int n_times) {
        /* Acquire GIL before calling Python code */
        py::gil_scoped_acquire acquire;

        PYBIND11_OVERRIDE_PURE(
            std::string, /* Return type */
            Animal,      /* Parent class */
            go,          /* Name of function */
            n_times      /* Argument(s) */
        );
    }
};

PYBIND11_MODULE(example, m) {
    py::class_<Animal, PyAnimal> animal(m, "Animal");
    animal
        .def(py::init<>())
        .def("go", &Animal::go);

    py::class_<Dog>(m, "Dog", animal)
        .def(py::init<>());

    m.def("call_go", [](Animal *animal) -> std::string {
        /* Release GIL before calling into (potentially long-running) C++ code */
        py::gil_scoped_release release;
        return call_go(animal);
    });
}
```

我们可以使用`call_guard`策略来简化`call_go`的封装：

```c++
m.def("call_go", &call_go, py::call_guard<py::gil_scoped_release>());
```
