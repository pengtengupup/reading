### item1. 模版类型推导

对于以下一个函数模板

```c_cpp
template<typename T>
void f(ParamType param)

f(expr);                        //使用表达式调用f
```

编译器会进行两次类型推导：推导param的类型与推导T的类型。我们可能很自然的期望T和传递进函数的实参是相同的类型，也就是，T为expr的类型。但有时情况并非总是如此，T的类型推导不仅取决于expr的类型，也取决于ParamType的类型。

- ParamType是一个指针或引用但不是通用引用时（即T&）：此时会忽略expr中的引用部分。
- ParamType是一个通用引用（T&&）：此时如果expr是左值引用，则T和ParamType都会推导为左值引用（特殊对待），如果expr是右值引用，则忽略引用推导为右值（同1）
- ParamType既不是指针也不是引用：按照传值的方式处理，会忽略expr中的const 与 引用部分。数组名或者函数名实参会退化为指针

### item2. auto类型推导

- auto类型推导通常和模板类型推导相同。但是auto类型推导假定花括号初始化代表std::initializer_list，而模板类型推导不这样做（在使用auto作为类型说明符的变量声明中，类型说明符代替了ParamType）
  ```
  auto x3 = { 27 };               //类型是std::initializer_list<int>，
  ```

- 在C++14中auto允许出现在函数返回值或者lambda函数形参中，但是它的工作机制是模板类型推导那一套方案，而不是auto类型推导

ps：关于初始化：

截至C++11标准，该语言中共有三种主要的创建对象的方式，分别为List Initialization, Direct Initialization和Copy Initialization。

其中列表初始化不允许隐式窄化转换。

### item3. decltype

- decltype总是不加修改的产生变量或者表达式的类型。
- 对于T类型的不是单纯的变量名的左值表达式，decltype总是产出T的引用即T&（decltype((x))是int&）。
- C++14支持decltype(auto)，就像auto一样，推导出类型，但是它使用decltype的规则进行推导。

常配合尾置返回类型使用：

```
template<typename Container, typename Index>    //可以工作，
auto authAndAccess(Container& c, Index i)       //但是需要改良
    ->decltype(c[i])
{
    authenticateUser();
    return c[i];
}
```

右值不能被绑定到左值引用上（除非这个左值引用是一个const（lvalue-references-to-const）

<br/>

### item21：优先使用make_unique和make_shared

std::shared_ptr<Widget> spw(new Widget);这段代码需要进行内存分配，但它实际上执行了两次（控制块+对象）。

auto spw = std::make_shared<Widget>();一次分配足矣。这是因为std::make_shared分配一块内存，同时容纳了Widget对象和控制块。

<br/>

### item31 避免使用默认捕获模式

使用lambda表达式需要注意出现悬空引用/悬空指针的问题，即捕获的对象/指针生命周期短于闭包的生命周期。

捕获只能应用于lambda被创建时所在作用域里的non-static局部变量（包括形参）。

每一个non-static成员函数都有一个this指针，每次你使用一个类内的数据成员时都会使用到这个指针。例如，在任何Widget成员函数中，编译器会在内部将divisor替换成this->divisor。
