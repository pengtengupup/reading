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

### item7 区分使用()或者{}创建对象

一般来说，初始化值要用圆括号()或者花括号{}括起来，或者放到等号"="的右边：

注意在初始化中使用"="没有发生赋值运算，实际上调用的是构造函数！

```c_cpp
Widget w1;              //调用默认构造函数
Widget w2 = w1;         //不是赋值运算，调用拷贝构造函数
w1 = w2;                //是赋值运算，调用拷贝赋值运算符（copy operator=）
```

- 花括号初始化是最广泛使用的初始化语法，它防止变窄转换，并且对于C++最令人头疼的解析有天生的免疫性（即无法区分TimeKeeper time_keeper(Timer());是函数声明还是定义变量，编译器默认会看做函数声明），这在调用无参构造函数时会出现。解决方案可以采用大括号初始化或者使用额外的括号或拷贝初始化。
  ```c_cpp
  TimeKeeper time_keeper( /*Avoid MVP*/ (Timer()) );
  TimeKeeper time_keeper = TimeKeeper(Timer());
  ```

- 使用花括号初始化的问题在于编译器会尽最大努力将括号初始化与std::initializer_list参数匹配，即便其他构造函数看起来是更好的选择
- 默认使用圆括号初始化的开发者主要被C++98语法一致性、避免std::initializer_list自动类型推导、避免不会不经意间调用std::initializer_list构造函数这些优点所吸引。这些开发者也承认有时候只能使用花括号（比如创建一个包含着特定值的容器）

注意初始化化对象时加()与不加（）的区别：

对于内置类型，加 ()会使用值初始化，不加 ()不进行初始化，内存中数据是随机的。

对于自定义类型：都是调用默认构造函数。

### item8 优先使用nullptr

优先考虑nullptr而非0和NULL

原因：

```
void f(int);        //三个f的重载函数
void f(bool);
void f(void*);

f(0);               //调用f(int)而不是f(void*)

f(NULL);            //可能不会被编译，一般来说调用f(int)，
                    //绝对不会调用f(void*)

f(nullptr);         //调用重载函数f的f(void*)版本
```

使用nullptr代替0和NULL可以避开了那些令人奇怪的函数重载决议。

### item9优先使用using别名而非typedef

- typedef不支持模板化，但是别名声明支持。

```c_cpp
template<typename T>                            //MyAllocList<T>是
using MyAllocList = std::list<T, MyAlloc<T>>;   //std::list<T, MyAlloc<T>>
                                                //的同义词

MyAllocList<Widget> lw;                         //用户代码

```

```c_cpp
template<typename T>                            //MyAllocList<T>是
struct MyAllocList {                            //std::list<T, MyAlloc<T>>
    typedef std::list<T, MyAlloc<T>> type;      //的同义词  
};

MyAllocList<Widget>::type lw;                   //用户代码
```

如果你想使用在一个模板内使用typedef声明一个链表对象，而这个对象又使用了模板形参，你就不得不在typedef前面加上typename。

原因在于这里MyAllocList<T>::type使用了一个类型，这个类型依赖于模板参数T。因此MyAllocList<T>::type是一个依赖类型，因为type是变量还是类型依赖于T。

- 别名模板避免了使用“::type”后缀，而且在模板中使用typedef还需要在前面加上typename
- C++14提供了C++11所有type traits转换的别名声明版本（如std::remove_reference_t）

### item 10 优先考虑限域enum而非非限域enum

非限域枚举：

```c_cpp
enum Color { black, white, red };   //black, white, red在
                                    //Color所在的作用域
auto white = false;                 //错误! white早已在这个作用
                                    //域中声明
```

限域枚举

```c_cpp
enum class Color { black, white, red }; //black, white, red
                                        //限制在Color域内
auto white = false;                     //没问题，域内没有其他“white”

Color c = white;                        //错误，域中没有枚举名叫white

Color c = Color::white;                 //没问题
auto c = Color::white;                  //也没问题（也符合Item5的建议）
```

优点：（1）减少命名空间污染。（2）枚举名是强类型，防止隐式转换为整形。

缺点：限域enum的枚举名仅在enum内可见。要转换为其它类型只能使用cast。

### item11 优先考虑delete函数而非使用未定义的私有声明

优点：

（1）deleted函数不能以任何方式被调用，即使你在成员函数或者友元函数里面调用deleted函数也不能通过编译。

（2）deleted函数还有一个重要的优势是任何函数都可以标记为deleted（包括非成员函数和模板实例），而只有成员函数可被标记为private。

```c_cpp
bool isLucky(int number);       //原始版本
bool isLucky(char) = delete;    //拒绝char
bool isLucky(bool) = delete;    //拒绝bool
bool isLucky(double) = delete;  //拒绝float和double
```

### item12 使用override声明重写函数

好处：让编译器帮助检查是否符合重写条件。

成员函数引用限定让我们可以区别对待左值对象和右值对象（即*this)

```c_cpp
class Widget {
public:
    using DataType = std::vector<double>;
    …
    DataType& data() &              //对于左值Widgets,
    { return values; }              //返回左值
    
    DataType data() &&              //对于右值Widgets,
    { return std::move(values); }   //返回右值
    …

private:
    DataType values;
};

```

重写的条件：

基类函数必须是virtual
基类和派生类函数名必须完全一样（除非是析构函数)
基类和派生类函数形参类型必须完全一样
基类和派生类函数常量性constness必须完全一样
基类和派生类函数的返回值和异常说明（exception specifications）必须兼容

函数的引用限定符（reference qualifiers）必须完全一样。（C11）

### item13 优先考虑const_iterator而非iterator

优先考虑const_iterator而非iterator
在最大程度通用的代码中，优先考虑非成员函数版本的begin，end，rbegin等，而非同名成员函数

### item14：如果函数不抛出异常请使用noexcept

给不抛异常的函数加上noexcept的动机：它允许编译器生成更好的目标代码。

如果在运行时，f出现一个异常，那么就和f的异常说明冲突了。在C++98的异常说明中，调用栈（the call stack）会展开至f的调用者，在一些与这地方不相关的动作后，程序被终止。C++11异常说明的运行时行为有些不同：调用栈只是可能在程序终止前展开。

展开调用栈和可能展开调用栈两者对于代码生成（code generation）有非常大的影响。在一个noexcept函数中，当异常可能传播到函数外时，优化器不需要保证运行时栈（the runtime stack）处于可展开状态；也不需要保证当异常离开noexcept函数时，noexcept函数中的对象按照构造的反序析构。而标注“throw()”异常声明的函数缺少这样的优化灵活性，没加异常声明的函数也一样。

```c_cpp
int f(int x) throw();   //C++98风格，没有来自f的异常
int f(int x) noexcept;  //C++11风格，没有来自f的异常
```

例如std::vector::push_back受益于“如果可以就移动，如果必要则复制”策略，对于这个函数只有在知晓移动不抛异常的情况下用C++11的移动操作替换C++98的复制操作才是安全的（基于移动构造函数是否有noexcept）。

noexcept对于移动语义，swap，内存释放函数和析构函数非常有用

### item15：尽可能的使用constexpr

constexpr表明一个值不仅仅是常量，还是编译期可知的。

const 保证变量在初始化后不可修改（**只读语义**），但其值可能在运行时确定；而 constexpr 确保变量是一个编译期常量，它的值必须在编译时就能确定，它比 const 更强调“常量”语义。

但是当作用于函数时，constexpr函数的实参在编译期可知，那么结果将在编译期计算；当一个constexpr函数被一个或者多个编译期不可知值调用时，它就像普通函数一样，运行时计算它的结果。

而检测constexpr函数是否产生编译时期值的方法很简单，就是利用std::array需要编译期常值才能编译通过的小技巧。

优点：使用constexpr关键字可以最大化你的对象和函数可以使用的场景。加上constexpr相当于宣称“我能被用在C++要求常量表达式的地方”。

### item16：让const成员函数线程安全

确保const成员函数线程安全，除非你确定它们永远不会在并发上下文（concurrent context）中使用。
使用std::atomic变量可能比互斥量提供更好的性能，但是它只适合操作单个变量或内存位置。

### item17 特殊成员函数的生成

在C++术语中，特殊成员函数是指C++自己生成的函数。C++98有四个：默认构造函数，析构函数，拷贝构造函数，拷贝赋值运算符。

这些函数仅在需要的时候才生成，比如某个代码使用它们但是它们没有在类中明确声明。默认构造函数仅在类完全没有构造函数的时候才生成。

生成的特殊成员函数是隐式public且inline，它们是非虚的，除非相关函数是在派生类中的析构函数，派生类继承了有虚析构函数的基类。在这种情况下，编译器为派生类生成的析构函数是虚的。

C++11特殊成员函数俱乐部迎来了两位新会员：移动构造函数和移动赋值运算符。

```c_cpp
class Widget {
public:
    …
    Widget(Widget&& rhs);               //移动构造函数
    Widget& operator=(Widget&& rhs);    //移动赋值运算符
    …
};
```

移动操作仅在需要的时候生成，如果生成了，就会对类的non-static数据成员执行逐成员的移动。对不可移动类型（即对移动操作没有特殊支持的类型，比如大部分C++98传统类）使用“移动”操作实际上执行的是拷贝操作。逐成员移动的核心是**对对象使用std::move**，然后函数决议时会选择执行移动还是拷贝操作。

- 两个拷贝操作是独立的：声明一个不会限制编译器生成另一个。
- 两个移动操作不是相互独立的。如果你声明了其中一个，编译器就不再生成另一个。
- 如果一个类显式声明了拷贝操作，编译器就不会生成移动操作。这种限制的解释是如果声明拷贝操作（构造或者赋值）就暗示着平常拷贝对象的方法（逐成员拷贝）不适用于该类，编译器会明白如果逐成员拷贝对拷贝操作来说不合适，逐成员移动也可能对移动操作来说不合适。
- 声明移动操作（构造或赋值）使得编译器禁用自动生成拷贝操作。（编译器通过给拷贝操作加上delete来保证）毕竟，如果逐成员移动对该类来说不合适，也没有理由指望逐成员拷贝操作是合适的。
- Rule of Three：如果你声明了拷贝构造函数，拷贝赋值运算符，或者析构函数三者之一，你应该也声明其余两个。来源与经验，即用户声明拷贝源于该类有其他资源的管理，因此通常涉及析构。
- Rule of Three暗示的规则，使得C++11**不会为那些有用户定义的析构函数的类生成移动操作**（声明析构不会生成移动）。

所以仅当下面条件成立时才会生成移动操作（当需要时）：

类中没有拷贝操作
类中没有移动操作
类中没有用户定义的析构

== 总结==

- 默认构造函数：和C++98规则相同。仅当类不存在用户声明的构造函数时才自动生成。
- 析构函数：基本上和C++98相同；稍微不同的是现在析构默认noexcept。和C++98一样，仅当基类析构为虚函数时该类析构才为虚函数。
- 拷贝构造函数：和C++98运行时行为一样：逐成员拷贝non-static数据。仅当类没有用户定义的拷贝构造时才生成。如果类声明了移动操作它就是delete的。当用户声明了拷贝赋值或者析构，该函数自动生成已被废弃（C++11抛弃了已声明拷贝操作或析构函数的类的拷贝操作的自动生成）。
- 拷贝赋值运算符：和C++98运行时行为一样：逐成员拷贝赋值non-static数据。仅当类没有用户定义的拷贝赋值时才生成。如果类声明了移动操作它就是delete的。当用户声明了拷贝构造或者析构，该函数自动生成已被废弃。
- 移动构造函数和移动赋值运算符：都对非static数据执行逐成员移动。仅当类没有用户定义的拷贝操作，移动操作或析构时才自动生成。

## 智能指针

### item18：对于独占资源使用std::unique_ptr

std::unique_ptr是轻量级、快速的、只可移动（move-only）的管理专有所有权语义资源的智能指针。

默认情况，资源销毁通过delete实现，但是支持自定义删除器。有状态的删除器和函数指针会增加std::unique_ptr对象的大小。

将std::unique_ptr转化为std::shared_ptr非常简单（可以直接赋值）

实现工厂函数：

```c_cpp
auto delInvmt = [](Investment* pInvestment)         //自定义删除器
                {                                   //（lambda表达式）
                    makeLogEntry(pInvestment);
                    delete pInvestment; 
                };

template<typename... Ts>
std::unique_ptr<Investment, decltype(delInvmt)>     //更改后的返回类型
makeInvestment(Ts&&... params)
{
    std::unique_ptr<Investment, decltype(delInvmt)> //应返回的指针
        pInv(nullptr, delInvmt);
    if (/*一个Stock对象应被创建*/)
    {
        pInv.reset(new Stock(std::forward<Ts>(params)...));
    }
    else if ( /*一个Bond对象应被创建*/ )   
    {     
        pInv.reset(new Bond(std::forward<Ts>(params)...));   
    }   
    else if ( /*一个RealEstate对象应被创建*/ )   
    {     
        pInv.reset(new RealEstate(std::forward<Ts>(params)...));   
    }   
    return pInv;
}

```

### item19：对于共享资源使用std::shared_ptr

注意向shared_ptr构造函数一个原始指针，它会为指向的对象创建一个控制块，因此重复传入同一个原始指针会造成重复delete问题！

避免传给std::shared_ptr构造函数原始指针。通常替代方案是使用std::make_shared。

如果你必须传给std::shared_ptr构造函数原始指针，直接传new出来的结果，不要传指针变量。

性能开销：

- std::shared_ptr大小是原始指针的两倍。（指向控制块的指针+指向资源的原始指针）
- 引用计数（控制块）的内存必须动态分配，除非使用make_shared。
- 递增递减引用计数必须是原子性的。

### item20：当std::shared_ptr可能悬空时使用std::weak_ptr

std::weak_ptr通常从std::shared_ptr上创建。当从std::shared_ptr上创建std::weak_ptr时两者指向相同的对象，但是std::weak_ptr不会影响所指对象的引用计数。

悬空的std::weak_ptr被称作已经expired（过期）。

```
auto spw =                      //spw创建之后，指向的Widget的
    std::make_shared<Widget>(); //引用计数（ref count，RC）为1。
                                //std::make_shared的信息参见条款21
…
std::weak_ptr<Widget> wpw(spw); //wpw指向与spw所指相同的Widget。RC仍为1
…
spw = nullptr;                  //RC变为0，Widget被销毁。
                                //wpw现在悬空
if (wpw.expired()) …            //如果wpw没有指向对象…

```

将检查和解引用分开会引入竞态条件：在调用expired和解引用操作之间，另一个线程可能对指向这对象的std::shared_ptr重新赋值或者析构，并由此造成对象已析构。

因此我们需要的是一个原子操作检查std::weak_ptr是否已经过期，如果没有过期就访问所指对象。这可以通过从std::weak_ptr创建std::shared_ptr来实现：

一种形式是std::weak_ptr::lock，它返回一个std::shared_ptr，如果std::weak_ptr过期这个std::shared_ptr为空：

另一种形式是以std::weak_ptr为实参构造std::shared_ptr。这种情况中，如果std::weak_ptr过期，会抛出一个异常：

应用：

（1）缓存：缓存对象的指针需要知道它是否已经悬空，因为当工厂客户端使用完工厂产生的对象后，对象将被销毁，关联的缓存条目会悬空。所以缓存应该使用std::weak_ptr，这可以知道是否已经悬空。

```c_cpp
std::shared_ptr<const Widget> fastLoadWidget(WidgetId id)
{
  static std::unordered_map<WidgetId, std::weak_ptr<const Widget>> cacahe;
  auto objPtr = cache[id].lock();
  if (!objPtr) {
    objPtr = LoadWidget(id);
    cache[id] = objPtr;
  }
  return objPtr;
}
```

(2) 解决循环引用：A和B都互相持有对方的std::shared_ptr，导致的std::shared_ptr环状结构（A指向B，B指向A）阻止A和B的销毁。甚至A和B无法从其他数据结构访问了。可以另B保留指向A的weak_ptr。

（3）观察者设计模式（Observer design pattern）。一个合理的设计是每个subject持有一个std::weak_ptrs容器指向observers，因此可以在使用前检查是否已经悬空。

### item21：优先考虑make_unique和make_shared

和直接使用new相比，make函数消除了代码重复，提高了异常安全性。对于std::make_shared和std::allocate_shared，生成的代码更小更快。

不适合使用make函数的情况包括需要指定自定义删除器和希望用花括号初始化。

对于std::shared_ptr，其他不建议使用make函数的情况包括(1)有自定义内存管理的类；(2)特别关注内存的系统，非常大的对象，以及std::weak_ptrs比对应的std::shared_ptrs活得更久。

### item22：当使用Pimpl惯用法，请在实现文件中定义特殊成员函数

一个已经被声明，却还未被实现的类型，被称为不完整类型

```c_cpp
class Widget                        //仍然在“widget.h”中
{
public:
    Widget();
    ~Widget();                      //析构函数在后面会分析
    …
    
private:
    struct Impl;                    //声明一个 实现结构体
    Impl *pImpl;                    //以及指向它的指针
};
```

Pimpl惯用法通过减少在类实现和类使用者之间的编译依赖来减少编译时间。

注意如果对于std::unique_ptr类型的pImpl指针，需要在头文件的类里声明特殊的成员函数，但是在实现文件里面来实现他们（为了能够在析构/移动时得到完整类型实现，uinique_ptr的删除器是其一部分，需要在编译时生成delete的完整代码，需要类型是完整）。而shared_ptr删除器是在构造时动态存储的，允许在类定义中使用不完整类型，只要构造发生在完整类型可见的地方。

## 右值引用、移动语义和完美转发

### item23：理解std::move和std::forward

std::move和std::forward仅仅是执行转换（cast）的函数（事实上是函数模板）。std::move无条件的将它的实参转换为右值，而std::forward只在特定情况满足时下进行转换(它的实参用右值初始化时，转换为一个右值。)。

不要在你希望能移动对象的时候，声明他们为const。例子：

std::move(text)的结果是一个const std::string的右值。这个右值不能被传递给std::string的移动构造函数，因为移动构造函数只接受一个指向non-const的std::string的右值引用。然而，该右值却可以被传递给std::string的拷贝构造函数，因为lvalue-reference-to-const允许被绑定到一个const右值上。

<br/>

```c_cpp
void process(const Widget& lvalArg);        //处理左值
void process(Widget&& rvalArg);             //处理右值

template<typename T>                        //用以转发param到process的模板
void logAndProcess(T&& param)
{
    auto now =                              //获取现在时间
        std::chrono::system_clock::now();
    
    makeLogEntry("Calling 'process'", now);
    process(std::forward<T>(param)); // 因为param是函数形参，是左值
}

```

区别：std::move的使用代表着无条件向右值的转换，而使用std::forward只对绑定了右值的引用进行到右值转换。这是两种完全不同的动作。前者是典型地为了移动操作，而后者只是传递（亦为转发）一个对象到另外一个函数，保留它原有的左值属性或右值属性。

### item24：区分通用引用与右值引用

如果一个函数模板形参的类型为T&&，并且T需要被推导得知，或者如果一个对象被声明为auto&&，这个形参或者对象就是一个通用引用。

如果类型声明的形式不是标准的type&&，或者如果类型推导没有发生，那么type&&代表一个右值引用（即使一个简单的const修饰符的出现，也足以使一个引用失去成为通用引用的资格）。

通用引用的底层是引用折叠，如果它被右值初始化，就会对应地成为右值引用；如果它被左值初始化，就会成为左值引用。

“T&&”的另一种意思是，它既可以是右值引用，也可以是左值引用。此外，它们还可以绑定到const或者non-const的对象上，也可以绑定到volatile或者non-volatile的对象上，甚至可以绑定到既const又volatile的对象上。它们可以绑定到几乎任何东西。

### item25：对于右值引用使用move，通用引用使用forward

最后一次使用时，在右值引用上使用std::move，在通用引用上使用std::forward。
对按值返回的函数要返回的右值引用和通用引用，执行相同的操作。
如果局部对象可以被返回值优化消除，就绝不使用std::move或者std::forward。

返回值优化（RVO）：编译器可能会在按值返回的函数中消除对局部对象的拷贝（或者移动）。条件：（1）局部对象与函数返回值的类型相同；（2）局部对象就是要返回的东西

### item26：避免对通用引用函数进行重载

对通用引用形参的函数进行重载，通用引用函数的调用机会几乎总会比你期望的多得多。

完美转发构造函数是糟糕的实现，因为对于non-const左值，它们比拷贝构造函数而更匹配，而且会劫持派生类对于基类的拷贝和移动构造函数的调用。

```c_cpp
class SpecialPerson: public Person {
public:
    SpecialPerson(const SpecialPerson& rhs) //拷贝构造函数，调用基类的
    : Person(rhs)                           //完美转发构造函数！
    { … }

    SpecialPerson(SpecialPerson&& rhs)      //移动构造函数，调用基类的
    : Person(std::move(rhs))                //完美转发构造函数！
    { … }
};
```

### item27：重载通用引用的替代

通用引用和重载的组合替代方案包括使用不同的函数名，通过lvalue-reference-to-const传递形参，按值传递形参，使用tag dispatch。

通过std::enable_if约束模板，允许组合通用引用和重载使用，但它也控制了编译器在哪种条件下才使用通用引用重载。

```c_cpp
class Person {
public:
    template<                       //同之前一样
        typename T,
        typename = std::enable_if_t<
            !std::is_base_of<Person, std::decay_t<T>>::value
            &&
            !std::is_integral<std::remove_reference_t<T>>::value
        >
    >
    explicit Person(T&& n)
    : name(std::forward<T>(n))
    {
        //断言可以用T对象创建std::string
        static_assert(
        std::is_constructible<std::string, T>::value,
        "Parameter n can't be used to construct a std::string"
        );

        …               //通常的构造函数的工作写在这

    }
    
    …                   //Person类的其他东西（同之前一样）
};

```

### item28：引用折叠

规则：如果任一引用为左值引用，则结果为左值引用。否则（即，如果引用都是右值引用），结果为右值引用。

四种情况会发生引用折叠，模板实例化和auto的类型生成，typedef和别名声明、decltype使用的情况。

通用引用就是在特定上下文的右值引用，上下文是通过类型推导区分左值还是右值，并且发生引用折叠的那些地方。

### item二十九：假定移动操作不存在，成本高，未被使用

存在几种情况，C++11的移动语义并无优势：

没有移动操作：要移动的对象没有提供移动操作，所以移动的写法也会变成复制操作。

移动不会更快：要移动的对象提供的移动操作并不比复制速度更快（SSO、std::array）。

移动不可用：进行移动的上下文要求移动操作不会抛出异常，但是该操作没有被声明为noexcept。

### item三十：熟悉完美转发失败的情况

当模板类型推导失败或者推导出错误类型，完美转发会失败。

导致完美转发失败的实参种类有花括号初始化，作为空指针的0或者NULL，仅有声明的整型static const数据成员，模板和重载函数的名字，位域。

```
auto il = { 1, 2, 3 };  //il的类型被推导为std::initializer_list<int>
fwd(il);                //可以，完美转发il给f

```

### item31 避免使用默认捕获模式

使用lambda表达式需要注意出现悬空引用/悬空指针的问题，即捕获的对象/指针生命周期短于闭包的生命周期。

捕获只能应用于lambda被创建时所在作用域里的non-static局部变量（包括形参）。

每一个non-static成员函数都有一个this指针，每次你使用一个类内的数据成员时都会使用到这个指针。例如，在任何Widget成员函数中，编译器会在内部将divisor替换成this->divisor。

## Lambda

应用：STL中的“_if”算法（比如，std::find_if，std::remove_if，std::count_if等）、比较函数（比如，std::sort，std::nth_element，std::lower_bound等）、快速创建std::unique_ptr和std::shared_ptr的自定义删除器
线程API中条件变量的谓词、
lambda有利于即时的回调函数，接口适配函数和特定上下文中的一次性函数。

lambda表达式：

```c_cpp
std::find_if(container.begin(), container.end(),
             [](int val){ return 0 < val && val < 10; });   //译者注：本行高亮
```

闭包（enclosure）是lambda创建的运行时对象。依赖捕获模式，闭包持有被捕获数据的副本或者引用。在上面的std::find_if调用中，闭包是作为第三个实参在运行时传递给std::find_if的对象。
闭包类（closure class）是从中实例化闭包的类。每个lambda都会使编译器生成唯一的闭包类。lambda中的语句成为其闭包类的成员函数中的可执行指令。

### item31：避免使用默认捕获方式

C++11中有两种默认的捕获模式：**按引用捕获&和按值捕获=**。但默认按引用捕获模式可能会带来**悬空引用**（闭包的声明周期大于捕获的变量的声明周期）的问题，而默认按值捕获模式可能会诱骗你让你以为能解决悬空引用的问题（实际上并没有），还会让你以为你的闭包是独立的（事实上也不是独立的）。

```c_cpp
void addDivisorFilter()
{
    auto calc1 = computeSomeValue1();
    auto calc2 = computeSomeValue2();

    auto divisor = computeDivisor(calc1, calc2);

    filters.emplace_back(                               //危险！对divisor的引用
        [&](int value) { return value % divisor == 0; } //将会悬空！
    );
}
```

- 捕获只能应用于lambda被创建时所在作用域里的non-static局部变量（包括形参）
- 成员函数内lambda闭包的生命周期与Widget对象的关系，闭包内含有Widget的this指针的拷贝。
- lambda可能会**依赖**局部变量和形参（它们可能被捕获），还有静态存储生命周期（static storage duration）的对象。这些对象定义在全局空间或者命名空间，或者在类、函数、文件中声明为static。这些对象也能在lambda里使用，但**它们不能被捕获**!但默认按值捕获可能会因此误导你，让你以为捕获了这些变量。

### item32:使用初始化捕获移动对象（C++14）

目标：有时我们有一个只能被移动的对象（例如std::unique_ptr或std::future）要进入到闭包里。

用初始化捕获可以让你指定：

从lambda生成的闭包类中的数据成员名称；
初始化该成员的表达式

```c_cpp
class Widget {                          //一些有用的类型
public:
    …
    bool isValidated() const;
    bool isProcessed() const;
    bool isArchived() const;
private:
    …
};

auto pw = std::make_unique<Widget>();   //创建Widget；使用std::make_unique
                                        //的有关信息参见条款21

…                                       //设置*pw

auto func = [pw = std::move(pw)]        //使用std::move(pw)初始化闭包数据成员
            { return pw->isValidated()
                     && pw->isArchived(); };

```

注意=”左侧的作用域不同于右侧的作用域。左侧的作用域是闭包类，右侧的作用域和lambda定义所在的作用域相同。

C11中的替代方案：构建可调用对象后者使用std::bind

```c_cpp
auto func = std::bind([](const std::vector<double>& data){/*使用data*/},
                      std::move(data)
                      ) // 参数为右值，因此将data移动构造到绑定对象中
```

当“调用”bind对象（即调用其函数调用运算符）时，其存储的实参将传递到最初传递给std::bind的可调用对象。在此示例中，这意味着当调用func（bind对象）时，func中所移动构造的data副本将作为实参传递给std::bind中的lambda。

默认情况下，从lambda生成的闭包类中的operator()成员函数为const的(除非将lambda声明为mutable)。

- 无法移动构造一个对象到C++11闭包，但是可以将对象移动构造进C++11的bind对象。

- 在C++11中模拟移动捕获包括将对象移动构造进bind对象，然后通过传引用将移动构造的对象传递给lambda。
- 由于bind对象的生命周期与闭包对象的生命周期相同，因此可以将bind对象中的对象视为闭包中的对象。

<br/>

### item33:对于auto&&形参使用decltype以std::forward它

c14特性：lambda中形参可以使用auto关键字。

```c_cpp
auto f = [](auto&& x)
         { return func(normalize(std::forward<decltype(x)>(x))); };
```

假定上述func对于左值与右值不同，因此需要std::forward（如果一个左值实参被传给通用引用的形参，那么形参类型会变成左值引用）

### item34：优先考虑lambda而不是std::bind

c14：using namespace std::literals;      //对于C++14后缀

```
hours(1),second(1)->1std::literals::h, 1std::literals::s
```

```c_cpp
auto setSoundB =                            //“B”代表“bind”
    std::bind(setAlarm,
              steady_clock::now() + 1h,     //不正确！见下
              _1,
              30s);

```

//占位符中的数字映射到其在std::bind形参列表中的位置，调用setSoundB时的第一个实参会被传递进setAlarm，作为调用setAlarm的第二个实参。

std::bind总是拷贝它的实参，但是调用者可以使用引用来存储实参，这要通过应用std::ref到实参上实现。

与lambda相比，使用std::bind进行编码的代码可读性较低，表达能力较低，并且效率可能较低。

在C++11中，可以在两个受约束的情况下证明使用std::bind是合理的：

1. 移动捕获。C++11的lambda不提供移动捕获，但是可以通过结合lambda和std::bind来模拟。 
2. 多态函数对象。因为bind对象上的函数调用运算符使用完美转发，所以它可以接受任何类型的实参

<br/>

## 并发API

### item35：优先考虑基于任务的编程而非基于线程的编程

```c_cpp
//基于线程（thread-based）的方式：
int DoSomeWork();
std::thread t(DoSomeWork);

// 基于任务的编程
auto fut = astd::async(DoSomeWork); //线程管理责任交给了标准库的开发者
```

基于线程的编程方式需要手动的**线程耗尽(系统线程)**、资源超额、负责均衡、平台适配性管理。

std::thread 是C++执行过程的对象，并作为软件线程的句柄（handle）。如果开发者试图创建大于系统支持的线程数量，会抛出std::system_error异常。

std::async工作流程：先将异步操作用std::packaged_task包装起来，然后将异步结果放到std::promise中，外面再通过futrue.get/wait获取未来结果。

使用std::async的好处:

- 可以处理函数结果或异常。
- 提供了线程的创建策略：std::launch::asyn：在调用async就开始创建线程。std::launch::deferred：延迟加载方式创建线程。调用async时不创建线程，直到调用了future的get或者wait时才创建线程。默认策略：std::launch::async | std::launch::deferred，即由系统决定，可能异步也可能延迟。

什么时候使用thread：

- 需要访问非常基础的线程API。如对底层系统级线程API的访问，std::thread对象提供了native_handle的成员函数
- 需要实现C++并发API之外的线程技术，如线程池。

### item36：如果有异步的必要请指定std::launch::async

std::launch::async启动策略意味着f必须异步执行，即在不同的线程。

std::launch::deferred启动策略意味着f仅当在std::async返回的future上调用get或者wait时才执行。当get或wait被调用，f会同步执行，即调用方被阻塞，直到f运行结束。

std::async的默认启动策略是两者都有可能，这种灵活性允许std::async和标准库的线程管理组件承担线程创建和销毁的责任，避免资源超额，以及平衡负载。

但是，使用默认启动策略的std::async也有一些有趣的影响。

```
auto fut = std::async(f);   //使用默认启动策略运行f
```

无法预测f是否会与t并发运行，因为f可能被安排延迟运行。

无法预测f是否会在与某线程相异的另一线程上执行，这个某线程在fut上调用get或wait。

无法预测f是否执行，因为不能确保在程序每条路径上，都会不会在fut上调用get或者wait。

如果 std::future 被销毁前未调用 get() 或 wait()：
对于 std::launch::async：主线程会阻塞，等待异步任务完成！
对于 std::launch::deferred：任务永远不会执行

什么时候可以使用默认策略：

任务不需要和执行get或wait的线程并行执行。
读写哪个线程的thread_local变量没什么问题。
可以保证会在std::async返回的future上调用get或wait，或者该任务可能永远不会执行也可以接受。
使用wait_for或wait_until编码时考虑到了延迟状态。

<br/>

```c_cpp
long long parallel_sum(const std::vector<int>& v) {
    if (v.size() < 10000) return std::accumulate(v.begin(), v.end(), 0LL);
    
    auto mid = v.begin() + v.size() / 2;
    
    // 左半部分异步计算
    auto left_future = std::async(
        std::launch::async,
        [&v, mid] { return std::accumulate(v.begin(), mid, 0LL); }
    );
    
    // 右半部分当前线程计算
    auto right_sum = std::accumulate(mid, v.end(), 0LL);
    
    // 合并结果
    return left_future.get() + right_sum;
}
```

std::async的默认启动策略是异步和同步执行兼有的。
这个灵活性导致访问thread_locals的不确定性，隐含了任务可能不会被执行的意思，会影响调用基于超时的wait的程序逻辑。
如果异步执行任务非常关键，则指定std::launch::async。
