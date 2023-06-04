# Effective Modern C++

> **Note**
> 感谢[CnTransGroup](https://github.com/CnTransGroup)的翻译

## CHAPTER 1 Deducing Types

### 1. Understand template type deduction

- 以如下模板为例，T的类型推导不仅取决于expr的类型，也取决于ParamType的类型，这里有三种情况

  ```C++
  template<typename T>
  void f(ParamType param);

  f(expr); // 从expr中推导T和ParamType
  ```

  - 情景一：ParamType是一个指针或引用，但不是通用引用，即`T& param`（关于通用引用请参见条款24。在这里你只需要知道它存在，而且不同于左值引用和右值引用）
    - 在这种情况下，如果expr的类型是一个引用，忽略引用部分
    - 然后expr的类型与ParamType进行模式匹配来决定T
  - 情景二：ParamType是一个通用引用，即`T&& param`
    - 如果expr是左值，T和ParamType都会被推导为左值引用。这非常不寻常
      - 第一，这是模板类型推导中唯一一种T被推导为引用的情况
      - 第二，虽然看似ParamType被声明为右值引用类型，但是最后推导的结果是左值引用
    - 如果expr是右值，就使用正常的（也就是情景一）推导规则
  - 情景三：ParamType既不是指针也不是引用，即`T param`
    - 当ParamType既不是指针也不是引用时，通过传值（pass-by-value）的方式处理，和之前一样，如果expr的类型是一个引用，忽略这个引用部分
    - 如果忽略expr的引用性（reference-ness）之后，expr是一个const，那就再忽略const。如果它是volatile，也忽略volatile（volatile对象不常见，它通常用于驱动程序的开发中。关于volatile的细节请参见条款40）
    - 但是此处有一个特殊情况，考虑`const char* const ptr = "Hello"`，此时ParamType被推导为`const char*`
      - 这种情况，ptr自身的值会被传给形参，根据类型推导的第三条规则，ptr自身的常量性constness将会被省略，所以param是`const char*`，也就是一个可变指针指向const字符串。在类型推导中，这个指针指向的数据的常量性constness将会被保留，但是当拷贝ptr来创造一个新指针param时，ptr自身的常量性constness将会被忽略。

- 另一个需要注意的问题是数组实参的类型推导，虽然数组和指针有时可以互换，但是两者并不相同
  - 对于`T param`，因为数组形参会视作指针形参，所以T被推导为指针，而不是数组
  - 但是对于`T& param`，虽然函数不能声明形参为真正的数组，但是可以接受指向数组的引用，例如对于`const char name[] = "J. P. Briggs";`，T被推导为`const char[13]`，形参（对这个数组的引用）的类型则为`const char (&)[13]`

- 最后一个细节是函数实参，不只是数组会退化为指针，函数类型也会退化为一个函数指针，我们对于数组类型推导的全部讨论都可以应用到函数类型推导和退化为函数指针上来
  - 指向函数的指针和指向函数的引用，实际上没有什么不同，但是如果你知道数组退化为指针，你也会知道函数退化为指针。

### 2. Understand auto type deduction

- auto类型推导通常和模板类型推导相同，但是auto类型推导假定`{}`初始化代表`std::initializer_list`，而模板类型推导不这样做

- 在C++14中auto允许出现在函数返回值或者lambda函数形参中，但是它的工作机制是模板类型推导那一套方案，而不是auto类型推导

### 3. Understand decltype

- 通常，decltype会精确的告诉你你想要的结果，相比模板类型推导和auto类型推导，decltype只是简单的返回名字或者表达式的类型
  - decltype最主要的用途就是用于声明函数模板，而这个函数返回类型依赖于形参类型
  - 至于为什么不能单独使用auto，是因为函数返回类型中使用auto，编译器实际上是使用的模板类型推导的那套规则
    - 如果那样的话这里就会有一些问题，例如`operator[]`对于大多数T类型的容器会返回一个`T&`，但是在模板类型推导期间，表达式的引用性（reference-ness）会被忽略

- 但是对于T类型的不是单纯的变量名的**左值表达式**，decltype总是产出T的引用（`T&`），当使用`decltype(auto)`的时候一定要加倍的小心，在表达式中看起来无足轻重的细节将会影响到推导结果

  ```C++
  decltype(auto) f2()
  {
      int x = 0;
      return (x); //return表达式，decltype((x))是int&，所以f2返回int&
  }
  ```

### 4. Know how to view deduced types

- 类型推断可以从IDE看出，从编译器报错看出，从Boost TypeIndex库的使用看出

- 这些工具可能既不准确也无帮助，所以理解C++类型推导规则才是最重要的

## CHAPTER 2 auto

### 5. Prefer auto to explicit type declarations

- auto变量从初始化表达式中推导出类型，所以变量必须初始化

- 通常使用`std::function`存放lambda表达式产生的可调用对象时，auto是更好的选择
  - auto避免了语法冗长，切不需要重复写很多形参类型，auto声明的变量保存一个和闭包一样类型的（新）闭包，因此使用了与闭包相同大小存储空间
  - 但实例化`std::function`并声明对象将会有固定的大小，这个大小可能不足以存储一个闭包，这个时候`std::function`的构造函数将会在堆上面分配内存来存储，这就造成了使用`std::function`比auto声明变量会消耗更多的内存，并且通过具体实现我们得知通过`std::function`调用一个闭包几乎无疑比auto声明的对象调用要慢
  - 换句话说，`std::function`方法比auto方法要更耗空间且更慢，还可能有out-of-memory异常。并且比起写`std::function`实例化的类型来，使用auto要方便得多，在这场存储闭包的比赛中，auto无疑取得了胜利
  - C++14中甚至可以把lambda形参也使用auto

  ```C++
  auto derefLess =                                //C++14版本
    [](const auto& p1,                          //被任何像指针一样的东西
       const auto& p2)                          //指向的值的比较函数
    { return *p1 < *p2; };
  ```

- 通常auto还可以避免一些移植性和效率性的问题，也使得重构更方便，还能让你少打几个字。

### 6. Use the explicitly typed initializer idiom when auto deduces undesired types

- 作为一个通则，不可见的代理类通常不适用于auto，这样类型的对象的生命期通常不会设计为能活过一条语句，所以创建那样的对象你基本上就走向了违反程序库设计基本假设的道路
  - “Proxy”设计模式是软件设计这座万神庙中一直都存在的高级会员，一些典型的可见代理类如`std::shared_ptr`和`std::unique_ptr`，而典型的不可见代理类如`std::vector<bool>::reference`和`std::bitset::reference`

- 解决方案是强制使用一个不同的类型推导形式，这种方法我通常称之为显式类型初始器惯用法（the explicitly typed initialized idiom）
  - 显式类型初始器惯用法使用auto声明一个变量，然后对表达式强制类型转换（cast）得出你期望的推导结果，例如`auto x = static_cast<bool>(y);`

## CHAPTER 3 Moving to Modern C++

### 7. Distinguish () and {} when creating objects

- `{}`语法最广泛使用的初始化语法，能用于各种不同的上下文，它防止了隐式的变窄转换，而且对于C++最令人头疼的解析也天生免疫
  - 令人头疼的解析是指，C++规定任何可以被解析为一个声明的东西必须被解析为声明，这个规则的副作用使你可能想创建一个使用默认构造函数构造的对象，却不小心变成了函数声明

- 但是问题在于，在构造函数重载决议中，编译器会尽最大努力将`{}`初始化与`std::initializer_list`参数匹配，即便其他构造函数看起来是更好的选择
  - 例如对于数值类型的`std::vector`来说使用花括号初始化和圆括号初始化会造成巨大的不同
  - 而且在模板类选择使用圆括号初始化或使用花括号初始化创建对象是一个挑战

### 8. Prefer nullptr to 0 and NULL

- 优先考虑nullptr而非0和NULL
  - 使用nullptr代替0和NULL可以避开那些令人奇怪的函数重载决议（因为它们实际类型都是整型），也可以使代码表意明确
  - 使用nullptr，模板不会有什么特殊的转换，模板类型推导将0和NULL推导为一个错误的类型（即它们的实际类型，而不是作为空指针的隐含意义）

### 9. Prefer alias declarations to typedefs

- 优先使用别名声明而非typedef
  - typedef不支持模板化，但是别名声明支持。
  - 别名模板避免了使用“::type”后缀，而且在模板中使用typedef还需要在前面加上typename
  - C++14提供了C++11所有type traits转换的别名声明版本

### 10. Prefer scoped enums to unscoped enums

- 优先考虑限域enum（由于是通过`enum class`声明，所以也会被称为枚举类）而非未限域enum
  - 限域enum的枚举名仅在enum内可见，要转换为其它类型只能使用cast
  - 非限域/限域enum都支持底层类型说明语法，限域enum底层类型默认是int，非限域enum没有默认底层类型
  - 限域enum总是可以前置声明，非限域enum仅当指定它们的底层类型时才能前置声明

### 11. Prefer deleted functions to private undefined ones

- `= delete`和声明为私有成员可能看起来只是方式不同，别无其他区别，其实还有一些实质性意义差别的
  - deleted函数不能以任何方式被调用，即使你在成员函数或者友元函数里面调用deleted函数也不能通过编译
  - deleted函数被声明为public而不是private，是因为当客户端代码试图调用成员函数时，C++会在检查deleted状态前检查它的访问性，当调用一个私有的deleted函数，一些编译器只会给出该函数是private的错误
  - deleted函数还有一个重要的优势是任何函数都可以标记为deleted，而只有成员函数可被标记为private
    - 例如创建deleted重载函数，其参数就是我们想要过滤的类型，从而避免隐式类型转换的无意义调用
    - 另一个deleted函数用武之地是禁止一些模板的实例化，例如要求一个模板仅支持原生指针，则需要使用delete关键字标注`void*`（特殊情况，因为没办法对它们进行解引用，或者加加减减等）和`char*`（特殊情况，因为它们通常代表C风格的字符串，而不是正常意义下指向单个字符的指针）相关的模板实例

### 12. Declare overriding functions override

- C++11引入了两个上下文关键字（contextual keywords），`override`和`final`（向虚函数添加final可以防止派生类重写。final也能用于类，这时这个类不能用作基类），这两个关键字的特点是它们是保留的，它们只是位于特定上下文才被视为关键字

- C++11之后要想重写一个函数，必须满足下列全部需求，这么多的重写需求意味着哪怕一个小小的错误也会造成巨大的不同，所以最好使用override关键字来声明重写函数
  - 基类函数必须是virtual
  - 基类和派生类函数名必须完全一样（除非是析构函数）
  - 基类和派生类函数形参类型必须完全一样
  - 基类和派生类函数常量性constness必须完全一样
  - 基类和派生类函数的返回值和异常说明（exception specifications）必须兼容
  - 函数的引用限定符（reference qualifiers）必须完全一样，成员函数的引用限定符是C++11很少抛头露脸的特性，它可以限定成员函数只能用于左值或者右值。成员函数不需要virtual也能使用

    ```C++
    class Widget {
    public:
      using DataType = std::vector<double>;
      …
      DataType& data() &              //对于左值Widgets,
      { return values; }              //返回左值引用
      
      DataType data() &&              //对于右值Widgets,
      { return std::move(values); }   //返回右值（临时对象）
      …

    private:
      DataType values;
    };
    ```

### 13. Prefer const_iterators to iterators

- 优先考虑`const_iterator`而非`iterator`
  - 在最大程度通用的代码中，优先考虑非成员函数版本的begin，end，rbegin等，而非同名成员函数

### 14. Declare functions noexcept if they won't emit exceptions

- noexcept是函数接口的一部分，这意味着调用者可能会依赖它，noexcept可以影响到调用代码的异常安全性（exception safety）和效率，就其本身而言，函数是否为noexcept和成员函数是否const一样重要
  - 在一个noexcept函数中，当异常可能传播到函数外时，优化器不需要保证运行时栈（the runtime stack）处于可展开状态，也不需要保证当异常离开noexcept函数时，noexcept函数中的对象按照构造的反序析构，因此noexcept函数比non-noexcept函数更容易优化

- noexcept对于移动语义，swap，内存释放函数和析构函数非常有用，只要可能就应该将它们实现为noexcept
  - 例如在C++11中，`std::vector`在进行扩容时，一个很自然的优化就是将元素的复制操作替换为移动操作
    - 但是很不幸运，这会破坏push_back的异常安全保证，如果n个元素已经从老内存移动到了新内存区，但异常在移动第n+1个元素时抛出，那么push_back操作就不能完成
    - 但是原始的`std::vector`已经被修改：有n个元素已经移动走了，恢复std::vector至原始状态也不太可能，因为从新内存移动到老内存本身又可能引发异常
    - 这是个很严重的问题，因为老代码可能依赖于push_back提供的强烈的异常安全保证，因此C++11版本的实现不能简单的将push_back里面的复制操作替换为移动操作，除非知晓移动操作绝不抛异常，这时复制替换为移动就是安全的
  - 默认情况下，内存释放函数和析构函数（不管是用户定义的还是编译器生成的）都是隐式noexcept的，因此它们不需要声明noexcept
    - 析构函数非隐式noexcept的情况仅当类的数据成员（包括继承的成员还有继承成员内的数据成员）明确声明它的析构函数可能抛出异常（如声明`noexcept(false)`）
    - 这种析构函数不常见，标准库里面没有，如果一个对象的析构函数可能被标准库使用（比如在容器内或者被传给一个算法），析构函数又可能抛异常，那么程序的行为是未定义的

- 大多数函数是异常中立的，而不是noexcept，这些函数自己不抛异常，但是它们内部的调用可能抛出
  - 此时，异常中立函数允许那些抛出异常的函数在调用链上更进一步直到遇到异常处理程序，而不是就地终止
  - 异常中立函数绝不应该声明为noexcept，因为它们可能抛出那种“让它们过吧”的异常，因此大多数函数缺少noexcept设计

### 15. Use constexpr whenever possible

- constexpr对象是const，它被在编译期可知的值初始化
  - 编译期可知的值“享有特权”，它们可能被存放到只读存储空间中，对于那些嵌入式系统的开发者，这个特性是相当重要的
  - 更广泛的应用是“其值编译期可知”的常量整数会出现在需要“整型常量表达式（integral constant expression）的上下文中，这类上下文包括数组大小，整数模板参数（包括`std::array`对象的长度），枚举名的值，对齐修饰符（`alignas(val)`），等等
  - 简而言之，所有constexpr对象都是const，但不是所有const对象都是constexpr

- 涉及到constexpr函数时，constexpr对象的使用情况就更有趣了，当传递编译期可知的值时，constexpr函数可以产出编译期可知的结果
  - constexpr函数可以用于需求编译期常量的上下文，如果你传给constexpr函数的实参在编译期可知，那么结果将在编译期计算，如果实参的值在编译期不知道，你的代码就会被拒绝
  - 当一个constexpr函数被一个或者多个编译期不可知值调用时，它就像普通函数一样，运行时计算它的结果，这意味着你不需要两个函数，一个用于编译期计算，一个用于运行时计算，constexpr全做了

- 本条款的建议是尽可能的使用constexpr，因为constexpr对象和constexpr函数可以使用的范围比non-constexpr对象和函数大得多，使用constexpr关键字可以最大化你的对象和函数可以使用的场景，但是要注意的是constexpr是对象和函数接口的一部分

### 16. Make const member functions thread safe

- 确保const成员函数线程安全，除非你确定它们永远不会在并发上下文（concurrent context）中使用
  - 使用`std::atomic`变量可能比互斥量（`std::mutex`）提供更好的性能，但是它只适合操作单个变量或内存位置

### 17. Understand special member function generation

- 在C++术语中，特殊成员函数是指C++自己生成的函数

- C++98有四个：默认构造函数，析构函数，拷贝构造函数，拷贝赋值运算符
  - 当然在这里有些细则要注意，这些函数仅在需要的时候才生成，比如某个代码使用它们但是它们没有在类中明确声明
  - 默认构造函数仅在类完全没有构造函数的时候才生成（防止编译器为某个类生成构造函数，但是你希望那个构造函数有参数）
  - 生成的特殊成员函数是隐式public且inline，它们是非虚的，除非相关函数是在派生类中的析构函数，派生类继承了有虚析构函数的基类，在这种情况下，编译器为派生类生成的析构函数是虚的
  - C++11析构函数一个稍微不同的是现在析构默认noexcept

- C++11特殊成员函数俱乐部迎来了两位新会员：移动构造函数和移动赋值运算符
  - 移动操作仅在需要的时候生成，如果生成了，就会对类的non-static数据成员执行逐成员的移动，那意味着移动构造函数根据参数里面对应的成员移动构造出新的non-static部分，移动赋值运算符根据参数里面对应的non-static成员移动赋值
    - 移动构造函数也移动构造基类部分（如果有的话），移动赋值运算符也是移动赋值基类部分
    - 对一个数据成员或者基类使用移动构造或者移动赋值时，没有任何保证移动一定会真的发生，逐成员移动，实际上，更像是逐成员移动请求，因为对不可移动类型（即对移动操作没有特殊支持的类型，比如大部分C++98传统类）使用移动操作实际上执行的是拷贝操作
    - 逐成员移动的核心是对对象使用`std::move`，然后函数决议时会选择执行移动还是拷贝操作，可以简单记住如果支持移动就会逐成员移动类成员和基类成员，如果不支持移动就执行拷贝操作

- 由于C++11移动操作带来的变动，需要额外讨论移动操作生成的精确条件，以及对拷贝操作的影响
  - 众所周知，两个拷贝操作是独立的，声明一个不会限制编译器生成另一个，但是两个移动操作却不是相互独立的，如果你声明了其中一个，编译器就不再生成另一个
  - 因为如果给类声明了，比如，一个移动构造函数，就表明对于移动操作应怎样实现，与编译器应生成的默认逐成员移动有些区别
    - 如果逐成员移动构造有些问题，那么逐成员移动赋值同样也可能有问题，所以声明移动构造函数阻止移动赋值运算符的生成，声明移动赋值运算符同样阻止编译器生成移动构造函数
  - 再进一步，如果一个类显式声明了拷贝操作，编译器就不会生成移动操作
    - 这种限制的解释是如果声明拷贝操作（构造或者赋值）就暗示着平常拷贝对象的方法（逐成员拷贝）不适用于该类，编译器会明白如果逐成员拷贝对拷贝操作来说不合适，逐成员移动也可能对移动操作来说不合适
  - 再思考另一个方向，声明移动操作（构造或赋值）使得编译器禁用拷贝操作
    - 编译器通过给拷贝操作加上delete来保证，毕竟，如果逐成员移动对该类来说不合适，也没有理由指望逐成员拷贝操作是合适的，但是注意，禁用的是自动生成的拷贝操作，对于用户声明的拷贝操作不受影响
  - 还有一个需要讨论的是“Rule of Three”规则，此规则带来的后果就是只要出现用户定义的析构函数就意味着简单的逐成员拷贝操作不适用于该类
    - 那意味着如果一个类声明了析构，拷贝操作可能不应该自动生成，因为它们做的事情可能是错误的，所以有时C++11抛弃了已声明拷贝操作或析构函数的类的拷贝操作的自动生成
    - 这意味着如果你的某个声明了析构或者拷贝的类依赖自动生成的拷贝操作，你应该考虑升级这些类，消除依赖
      - 假设编译器生成的函数行为是正确的（即逐成员拷贝类non-static数据是你期望的行为），你的工作很简单，C++11的`= default`就可以表达你想做的
    - 好的，我知道你可能会问，到底什么是“Rule of Three”规则
      - 这个规则的概括便是如果声明了拷贝构造函数，拷贝赋值运算符，或者析构函数三者之一，你应该也声明其余两个
      - 它来源于长期的观察，即用户接管拷贝操作的需求几乎都是因为该类会做其他资源的管理，这也几乎意味着：
        - 无论哪种资源管理如果在一个拷贝操作内完成，也应该在另一个拷贝操作内完成
        - 类的析构函数也需要参与资源的管理（通常是释放）。通常要管理的资源是内存
      - 这也是为什么标准库里面那些管理内存的类（如会动态内存管理的STL容器）都声明了拷贝构造，拷贝赋值和析构

- 总结C++11对于特殊成员函数处理的规则如下：
  - 默认构造函数：和C++98规则相同，仅当类不存在用户声明的构造函数时才自动生成
  - 析构函数：基本上和C++98相同，稍微不同的是现在析构默认noexcept，和C++98一样，仅当基类析构为虚函数时该类析构才为虚函数
  - 拷贝构造函数：和C++98运行时行为一样，逐成员拷贝non-static数据，仅当类没有用户定义的拷贝构造时才生成，如果类声明了移动操作它就是delete的，当用户声明了拷贝赋值或者析构，该函数的自动生成已被废弃
  - 拷贝赋值运算符：和C++98运行时行为一样，逐成员拷贝赋值non-static数据，仅当类没有用户定义的拷贝赋值时才生成，如果类声明了移动操作它就是delete的，当用户声明了拷贝构造或者析构，该函数的自动生成已被废弃
  - 移动构造函数和移动赋值运算符：都对非static数据执行逐成员移动，仅当类没有用户定义的拷贝操作，移动操作或析构时才自动生成

- 另一个注意点是没有“成员函数模版阻止编译器生成特殊成员函数”的规则，这意味着此时编译器仍会生成移动和拷贝操作（假设正常生成它们的条件满足），即使可以模板实例化产出拷贝构造和拷贝赋值运算符的函数签名

## CHAPTER 4 Smart Pointers

### 18. Use std::unique_ptr for exclusive-ownership resource management

- `std::unique_ptr`是轻量级、快速的、只可移动（move-only）的管理专有所有权语义资源的智能指针
  - 默认情况，资源销毁通过delete实现，但是支持自定义删除器，有状态的删除器和函数指针会增加`std::unique_ptr`对象的大小
    - 无状态函数（stateless function）对象（比如不捕获变量的lambda表达式）对大小没有影响，这意味当自定义删除器可以实现为函数或者lambda时，尽量使用lambda
    - 无状态函数对象的大小为1，但是可以通过EBCO（Empty Base Class Optimisation）优化使其不占用空间
    - 所谓EBCO，在C++20之前通过继承空类来实现空间优化，C++20之后可以通过`[no_unique_address]`来让编译器检查空类并优化
    - 详见：[Empty Base Class Optimisation](https://www.cppstories.com/2021/no-unique-address/)
  - 将`std::unique_ptr`转化为`std::shared_ptr`非常简单

### 19. Use std::shared_ptr for shared-ownership resource management

- `std::shared_ptr`为有共享所有权的任意资源提供一种自动垃圾回收的便捷方式。
  - 但是引用计数暗示着性能问题
    - `std::shared_ptr`大小是原始指针的两倍，因为它内部包含一个指向资源的原始指针，还包含一个指向资源的引用计数值的原始指针
    - 引用计数的内存必须动态分配，条款21会解释使用`std::make_shared`创建`std::shared_ptr`可以避免引用计数的动态分配，但是还存在一些`std::make_shared`不能使用的场景，这时候引用计数就会动态分配
    - 递增递减引用计数必须是原子性的，因为多个reader、writer可能在不同的线程，原子操作通常比非原子操作要慢，所以即使引用计数通常只有一个word大小，你也应该假定读写它们是存在开销的。
  - 默认资源销毁是通过delete，但是也支持自定义删除器，删除器的类型是什么对于`std::shared_ptr`的类型没有影
    - 这种支持有别于`std::unique_ptr`，对于它来说，删除器类型是智能指针类型的一部分，但是对于`std::shared_ptr`则不是
    - 另一个不同于`std::unique_ptr`的地方是，指定自定义删除器不会改变`std::shared_ptr`对象的大小，不管删除器是什么，对象都是两个指针大小

- 回到刚才删除器的问题，自定义删除器可以是函数对象，函数对象可以包含任意多的数据，这意味着函数对象是任意大的，`std::shared_ptr`怎么能引用一个任意大的删除器而不使用更多的内存
  - 事实是它必须使用更多的内存，然而，那部分内存不是`std::shared_ptr`对象的一部分，那部分在堆上面，所在的数据结构通常叫做控制块（control block）
    - 每个`std::shared_ptr`管理的对象都有个相应的控制块，控制块除了包含引用计数值外还有一个自定义删除器的拷贝，当然前提是存在自定义删除器
    - 如果用户还指定了自定义分配器，控制块也会包含一个分配器的拷贝，控制块可能还包含一些额外的数据，如条款21提到的，一个次级引用计数weak count
  - 当指向对象的`std::shared_ptr`一创建，对象的控制块就建立了，通常，对于一个创建指向对象的`std::shared_ptr`的函数来说不可能知道是否有其他`std::shared_ptr`早已指向那个对象，所以控制块的创建会遵循下面几条规则
    - `std::make_shared`（参见条款21）总是创建一个控制块，它创建一个要指向的新对象，所以可以肯定`std::make_shared`调用时对象不存在其他控制块
    - 当从独占指针（`std::unique_ptr`）上构造出`std::shared_ptr`时会创建控制块，独占指针没有使用控制块，所以指针指向的对象没有关联控制块
    - 当从原始指针上构造出`std::shared_ptr`时会创建控制块，如果你想从一个早已存在控制块的对象上创建`std::shared_ptr`，你将假定传递一个`std::shared_ptr`或者`std::weak_ptr`（参见条款20）作为构造函数实参，而不是原始指针
  - 这些规则造成的后果就是从原始指针上构造超过一个`std::shared_ptr`就会让你走上未定义行为的快车道，因为指向的对象有多个控制块关联，多个控制块意味着多个引用计数值，多个引用计数值意味着对象将会被销毁多次

- 可以看出，`std::shared_ptr`给我们上了两堂课
  - 第一，避免传给`std::shared_ptr`构造函数原始指针，通常替代方案是使用`std::make_shared`，不过如果使用了自定义删除器，用`std::make_shared`就没办法做到了
  - 第二，如果必须传给`std::shared_ptr`构造函数原始指针，直接传`new`出来的结果，不要传指针变量

### 20. Use std::weak_ptr for std::shared_ptr-like pointers that can dangle

- 用`std::weak_ptr`替代可能会悬空的`std::shared_ptr`
  - `std::weak_ptr`的潜在使用场景包括：缓存、观察者列表、打破`std::shared_ptr`环状结构
  - 从效率角度来看，`std::weak_ptr`与`std::shared_ptr`基本相同
    - 两者的大小是相同的，使用相同的控制块（参见条款19），构造、析构、赋值操作涉及引用计数的原子操作
    - 这可能让你感到惊讶，因为我们知道`std::weak_ptr`不影响引用计数，但是其实是`std::weak_ptr`不参与对象的共享所有权，因此不影响指向对象的引用计数，实际上在控制块中还是有第二个引用计数，`std::weak_ptr`操作的是第二个引用计数

### 21. Prefer std::make_unique and std::make_shared to direct use of new

- 和直接使用new相比，make函数消除了代码重复，提高了异常安全性，对于`std::make_shared`和`std::allocate_shared`，生成的代码更小更快
  - `std::make_unique`和`std::make_shared`是三个make函数中的两个，它们接收任意的多参数集合，完美转发到构造函数去动态分配一个对象，然后返回这个指向这个对象的指针
  - 第三个make函数是`std::allocate_shared`，它行为和`std::make_shared`一样，只不过第一个参数是用来动态分配内存的allocator对象
  - 如果你对提高异常安全性有疑问，考虑`processWidget(std::shared_ptr<Widget>(new Widget), computePriority());`
    - 这段代码怎么会泄漏呢，答案和编译器将源码转换为目标代码有关，在运行时，一个函数的实参必须先被计算，这个函数再被调用，但是编译器不需要按照执行顺序生成代码
    - `new Widget`必须在`std::shared_ptr`的构造函数被调用前执行，因为`new`出来的结果作为构造函数的实参，但`computePriority`可能在这之前，之后，或者之间执行
    - 如果`computePriority`在之间被执行，一旦该函数抛出异常，那么第一步动态分配的`Widget`就会泄漏，因为它永远都不会被第三步的`std::shared_ptr`所管理了
    - 而使用`processWidget(std::make_shared<Widget>(), computePriority());`可以防止这种问题
  - 至于`std::make_shared`和`std::allocate_shared`生成更小，更快的代码，并使用更简洁的数据结构
    - 这是因为直接使用`new`需要两次内存分配，一次是为对象分配内存，一次是为控制块分配内存
    - 而`std::make_shared`和`std::allocate_shared`只需要一次内存分配，同时容纳了对象和控制块，这种优化减少了程序的静态大小，因为代码只包含一个内存分配调用，并且它提高了可执行代码的速度，因为内存只分配一次
    - 此外，使用`std::make_shared`避免了对控制块中的某些簿记信息的需要，潜在地减少了程序的总内存占用

- 不适合使用make函数（`std::unique_ptr`只有这两种情况，但是`std::shared_ptr`更多）的情况包括需要指定自定义删除器和希望用花括号初始化
  - 这意味着在make函数中，完美转发使用小括号，而不是花括号，因此如果你想用花括号初始化指向的对象，你必须直接使用new
  - 但是，条款30介绍了一个变通的方法，使用auto类型推导从花括号初始化创建`std::initializer_list`对象，然后将auto创建的对象传递给make函数

- 对于`std::shared_ptrs`，其他不建议使用make函数的情况包括更多
  - 有自定义内存管理的类，例如一些类重载了`operator new`和`operator delete`
    - 这些函数的存在意味着对这些类型的对象的全局内存分配和释放是不合常规的，设计这种定制操作往往只会精确的分配、释放对象大小的内存，例如，`Widget`类的`operator new`和`operator delete`只会处理`sizeof(Widget)`大小的内存块的分配和释放
    - 这种系列行为不太适用于`std::shared_ptr`对自定义分配（通过`std::allocate_shared`）和释放（通过自定义删除器）的支持，因为`std::allocate_shared`需要的内存总大小不等于动态分配的对象大小，还需要再加上控制块大小
    - 因此，使用make函数去创建重载了`operator new`和`operator delete`类的对象是个典型的糟糕想法
  - 特别关注内存的系统，非常大的对象，以及`std::weak_ptrs`比对应的`std::shared_ptrs`活得更久
    - 正如之前所说，控制块除了引用计数，还包含簿记信息，引用计数追踪有多少`std::shared_ptrs`指向控制块，但控制块还有第二个计数，记录多少个`std::weak_ptrs`指向控制块（即weak count，实际上，weak count的值不总是等于指向控制块的std::weak_ptr的数目）
    - 当一个`std::weak_ptr`检测它是否过期时，它会检测指向的控制块中的引用计数（而不是weak count），如果引用计数是0，`std::weak_ptr`就已经过期
    - 所以只要`std::weak_ptrs`引用一个控制块，该控制块必须继续存在，包含它的内存就必须保持分配，通过`std::shared_ptr`的make函数分配的内存，直到最后一个`std::shared_ptr`和最后一个指向它的`std::weak_ptr`已被销毁，才会释放
    - 如果对象类型非常大，而且销毁最后一个`std::shared_ptr`和销毁最后一个`std::weak_ptr`之间的时间很长，那么在销毁对象和释放它所占用的内存之间可能会出现延迟

### 22. When using the Pimpl Idiom, define special member functions in the implementation file

- Pimpl惯用法通过减少在类实现和类使用者之间的编译依赖来减少编译时间。
  - 对于`std::unique_ptr`类型的pImpl指针，需要在头文件的类里声明特殊的成员函数，但是在实现文件里面来实现他们，即使是编译器自动生成的代码可以工作，也要这么做，但此规则不适用于`std::shared_ptr`
  - `std::unique_ptr`和`std::shared_ptr`在pImpl指针上的表现上的区别的深层原因在于，他们支持自定义删除器的方式不同。
    - 对`std::unique_ptr`而言，删除器的类型是这个智能指针的一部分，这让编译器有可能生成更小的运行时数据结构和更快的运行代码，这种更高效率的后果之一就是`std::unique_ptr`指向的类型，在编译器的生成特殊成员函数（如析构函数，移动操作）被调用时，必须已经是一个完成类型
    - 而对`std::shared_ptr`而言，删除器的类型不是该智能指针的一部分，这让它会生成更大的运行时数据结构和稍微慢点的代码，但是当编译器生成的特殊成员函数被使用的时候，指向的对象不必是一个完成类型

## CHAPTER 5 RValue References, Move Semantics, and Perfect Forwarding

### 23. Understand std::move and std::forward

- 当你第一次了解到移动语义（move semantics）和完美转发（perfect forwarding）的时候，它们看起来非常直观
  - 移动语义使编译器有可能用廉价的移动操作来代替昂贵的拷贝操作，正如拷贝构造函数和拷贝赋值操作符给了你控制拷贝语义的权力，移动构造函数和移动赋值操作符也给了你控制移动语义的权力，移动语义也允许创建只可移动（move-only）的类型，例如`std::unique_ptr`，`std::future`和`std::thread`
  - 完美转发使接收任意数量实参的函数模板成为可能，它可以将实参转发到其他的函数，使目标函数接收到的实参与被传递给转发函数的实参保持一致
  - 而右值引用是连接这两个截然不同的概念的胶合剂，它是使移动语义和完美转发变得可能的基础语言机制

- 另一个需要牢记的一点是形参永远是左值，即使它的类型是一个右值引用
  - 比如`void f(Widget&& w);`，形参w是一个左值，即使它的类型是一个rvalue-reference-to-Widget

- 对于`std::move`，需要记住两点
  - 第一，不要在你希望能移动对象的时候，声明他们为const，对const对象的移动请求会悄无声息的被转化为拷贝操作
    - 这是因为移动构造函数只接受一个指向non-const的的右值引用，然而，该右值却可以被传递给拷贝构造函数，因为lvalue-reference-to-const允许被绑定到一个const右值上
    - 因此，新对象在成员初始化的过程中调用了拷贝构造函数，即使原对象已经被转换成了右值，这样是为了确保维持const属性的正确性
  - 第二，`std::move`不仅不移动任何东西，而且它也不保证它执行转换的对象可以被移动，关于`std::move`，你能确保的唯一一件事就是将它应用到一个对象上，能够得到一个右值

- 对于`std::forward`，只有当它的参数被绑定到一个右值时，才将参数转换为右值
  - 还记得函数的形参永远是左值吗，所以我们才需要一种机制，当且仅当传递给函数的用以初始化形参的实参是一个右值时，形参会被转换为一个右值

### 24. Distinguish universal references from rvalue references

- 通用引用的基础是一个“抽象”，其底层真相被称为引用折叠（reference collapsing），条款28将致力于讨论它，此处你只要能够区分通用引用即可
  - 如果一个函数模板形参的类型为`T&&`，并且T需要被推导得知，或者如果一个对象被声明为`auto&&`，这个形参或者对象就是一个通用引用
  - 如果类型声明的形式不是标准的`type&&`，或者如果**类型推导没有发生**，那么`type&&`代表一个右值引用
    - 即使一个简单的const修饰符的出现，也足以使一个引用失去成为通用引用的资格
  - 通用引用，如果它被右值初始化，就会对应地成为右值引用，如果它被左值初始化，就会成为左值引用

### 25. Use std::move on rvalue references, std::forward on universal references

- 当把右值引用转发给其他函数时，右值引用应该被无条件转换为右值（通过`std::move`），因为它们总是绑定到右值，当转发通用引用时，通用引用应该有条件地转换为右值（通过`std::forward`），因为它们只是有时绑定到右值
  - 但是需要注意，在有些稀少的情况下，你需要调用`std::move_if_noexcept`代替`std::move`（参考条款14）
  - 如果你在按值返回的函数中，返回值绑定到右值引用或者通用引用上，需要对返回的引用使用`std::move`或者`std::forward`，参考如下代码

    ```C++
    Matrix operator+(Matrix&& lhs, const Matrix& rhs)
    {
        lhs += rhs;
        return std::move(lhs); //移动lhs到返回值中，否则lhs是个左值的事实，会强制编译器拷贝它到返回值的内存空间
    }
    ```

  - 但是如果局部对象可以被返回值优化消除，就绝不要使用`std::move`或者`std::forward`
    - 编译器可能会在按值返回的函数中消除对局部对象的拷贝（或者移动），如果满足（1）局部对象与函数返回值的类型相同，（2）局部对象就是要返回的东西（适合的局部对象包括大多数局部变量，但函数形参不满足要求）
    - 函数的传值形参虽然没资格参与函数返回值的拷贝消除，但是如果作为返回值的话编译器会将其视作右值

### 26. Avoid overloading on universal references

- 使用通用引用的函数在C++中是最贪婪的函数，它们几乎可以精确匹配任何类型的实参（极少不适用的实参在条款30中介绍），这也是把重载和通用引用组合在一块是糟糕主意的原因，通用引用的实现会匹配比开发者预期要多得多的实参类型
  - 尤其是完美转发构造函数更是糟糕的实现，因为对于non-const左值，它们比拷贝构造函数更匹配，而且会劫持派生类对于基类的拷贝和移动构造函数的调用

### 27. Familiarize yourself with alternatives to overloading on universal references

- 通用引用和重载的组合替代方案包括使用不同的函数名，通过lvalue-reference-to-const传递形参，按值传递形参，使用tag dispatch

- 通过`std::enable_if`约束模板，允许组合通用引用和重载使用，但它也控制了编译器在哪种条件下才使用通用引用重载。
  - 通用引用参数通常具有高效率的优势，但是可用性就值得斟酌
  - 例如如下代码，所需要解决的是（1）加入一个Person构造函数重载来处理整型参数，（2）约束模板构造函数使其对于某些实参禁用

  ```C++
  class Person {
  public:
      template<
          typename T,
          typename = std::enable_if_t<
              !std::is_base_of<Person, std::decay_t<T>>::value
              &&
              !std::is_integral<std::remove_reference_t<T>>::value
          >
      >
      explicit Person(T&& n)          //对于std::strings和可转化为
      : name(std::forward<T>(n))      //std::strings的实参的构造函数
      { … }

      explicit Person(int idx)        //对于整型实参的构造函数
      : name(nameFromIdx(idx))
      { … }

      …                               //拷贝、移动构造函数等

  private:
      std::string name;
  };
  ```

### 28. Understand reference collapsing

- 引用折叠发生在四种情况下：模板实例化，auto类型推导，typedef与别名声明的创建和使用，以及decltype
  - 当编译器在引用折叠环境中生成了引用的引用时，结果就是单个引用
    - 有左值引用折叠结果就是左值引用，否则就是右值引用
  - 通用引用就是在特定上下文的右值引用，上下文就是指通过类型推导来区分左值和右值并发生引用折叠的地方

### 29. Assume that move operations are not present, not cheap, and not used

- 存在几种情况，C++11的移动语义并无优势：
  - 没有移动操作：要移动的对象没有提供移动操作，所以移动的写法也会变成复制操作
  - 移动不会更快：要移动的对象提供的移动操作并不比复制速度更快
  - 移动不可用：进行移动的上下文要求移动操作不会抛出异常，但是该操作没有被声明为noexcept
  - 值得一提的是，还有另一个场景，会使得移动并没有那么有效率
    - 源对象是左值：除了极少数的情况外（例如条款25），只有右值可以作为移动操作的来源
  - 上诉情况就是通用代码中的典型情况，比如编写模板代码，因为你不清楚你处理的具体类型是什么，因此可能需要假定移动操作不存在，成本高，未被使用，但是在已知的类型或者支持移动语义的代码中，就不需要上面的假设了

### 30. Familiarize yourself with perfect forwarding failure cases

- 当模板类型推导失败或者推导出错误类型时，我们称之为完美转发会失败，导致完美转发失败的实参种类有以下几种
  - 花括号初始化
    - 问题在于，将花括号初始化传递给未声明为`std::initializer_list`的函数模板形参，被判定为“非推导上下文”，但是auto面对这种情况的类型推导是成功的
  - 0或者NULL作为空指针，会使类型推导出错
  - 仅有声明的整型static const数据成员
  - 重载函数的名称和模板名称
    - 因为函数模板相比于普通函数是没有可接受的类型信息的，使得编译器不可能决定出哪个函数应被传递
  - 位域
    - 禁止的理由很充分，位域可能包含了机器字的任意部分（比如32位int的3-5位），但是这些东西无法直接寻址，在硬件层面引用和指针是一样的，所以没有办法创建一个指向任意bit的指针（C++规定你可以指向的最小单位是char），同样没有办法绑定引用到任意bit上

## CHAPTER 6 Lambda Expressions

### 31. Avoid default capture modes

- 与lambda相关的词汇可能会令人疑惑，让我们做一下简单的回顾
  - lambda表达式（lambda expression）就是一个表达式，例如`[](int val){ return 0 < val && val < 10; }`
  - 闭包（enclosure）是lambda创建的运行时对象，依赖捕获模式，闭包持有被捕获数据的副本或者引用，闭包是可作为实参在运行时传递给函数的对象，闭包通常可以拷贝，所以可能有多个闭包对应于一个lambda
  - 闭包类（closure class）是从中实例化闭包的类，每个lambda都会使编译器生成唯一的闭包类，lambda中的语句成为其闭包类的成员函数中的可执行指令

- C++11中有两种默认的捕获模式：按引用捕获和按值捕获
  - 但默认按引用捕获模式可能会带来悬空引用的问题，而默认按值捕获模式可能会诱骗你让你以为能解决悬空引用的问题（实际上并没有），还会让你以为你的闭包是独立的（事实上也不是独立的）
  - 例如lambda可能会依赖局部变量和形参（它们可能被捕获），还有静态存储生命周期（static storage duration）的对象，这些对象定义在全局空间或者命名空间，或者在类、函数、文件中声明为static
    - 这些对象虽然也能在lambda里使用，但它们不能被捕获，但默认按值捕获可能会因此误导你，让你以为捕获了这些变量

### 32. Use init capture to move objects into closures

- 使用C++14的初始化捕获将对象移动到闭包中
  - 初始化捕获可以让你指定从lambda生成的闭包类中的数据成员名称和初始化该成员的表达式

### 33. Use decltype on auto&& parameters to std::forward them

- C++14中泛型lambda（generic lambdas）是最值得期待的特性之一，在lambda的形参中可以使用auto关键字，再加上可变形参，意味着你可以实现如下代码

  ```C++
  auto f =
      [](auto&&... params)
      {
          return func(std::forward<decltype(params)>(params)...);
      };
  ```

- 但是要注意，对`auto&&`形参使用`decltype`以`std::forward`它们

### 34. Prefer lambdas to std::bind

- lambda几乎总是比`std::bind`更好的选择，因为lambda更易读，更具表达力并且可能更高效

## CHAPTER 7 The Concurrency API

### 35. Prefer task-based programming to thread-based

- 如果想要异步执行doAsyncWork函数，通常有两种方式，其一是通过创建`std::thread`执行doAsyncWork，这是应用了基于线程（thread-based）的方式，其二是将doAsyncWork传递给`std::async`，这是一种基于任务（task-based）的策略，传递给`std::async`的函数对象被称为一个任务（task）
  - 基于任务的方法通常比基于线程的方法更优，第一个原因是基于任务的方法代码量更少
  - 第二，假设调用doAsyncWork的代码对于其提供的返回值是有需求的，基于线程的方法对此无能为力，而基于任务的方法就简单了，因为`std::async`返回的`std::future`提供了get函数（从而可以获取返回值）
  - 第二，如果doAsycnWork发生了异常，get函数就显得更为重要，因为get函数可以提供抛出异常的访问，而基于线程的方法，如果doAsyncWork抛出了异常，程序会直接终止（通过调用`std::terminate`）

- 基于线程与基于任务最根本的区别在于，基于任务的抽象层次更高，基于任务的方式使得开发者从线程管理的细节中解放出来，对此在C++并发软件中总结了“thread”的三种含义
  - 硬件线程（hardware threads）是真实执行计算的线程，现代计算机体系结构为每个CPU核心提供一个或者多个硬件线程
  - 软件线程（software threads）（也被称为系统线程（OS threads、system threads））是操作系统管理的在硬件线程上执行的线程，通常可以存在比硬件线程更多数量的软件线程，因为当软件线程被阻塞的时候（比如 I/O、同步锁或者条件变量），操作系统可以调度其他未阻塞的软件线程执行提供吞吐量
  - `std::thread`是C++执行过程的对象，并作为软件线程的句柄（handle），有些`std::thread`对象代表“空”句柄，即没有对应软件线程，因为它们处在默认构造状态（即没有函数要执行），有些被移动走（移动到的`std::thread`就作为这个软件线程的句柄），有些被join（它们要运行的函数已经运行完），有些被detach（它们和对应的软件线程之间的连接关系被打断）

- 基于线程的编程方式需要手动的线程耗尽、资源超额、负责均衡、平台适配性管理，而基于任务的设计为开发者避免了手动线程管理的痛苦，并且自然提供了一种获取异步执行程序的结果（即返回值或者异常）的方式，但是，仍然存在一些场景直接使用`std::thread`会更有优势
  - 第一，需要访问非常基础的线程API，C++并发API通常是通过操作系统提供的系统级API（pthreads或者Windows threads）来实现的，系统级API通常会提供更加灵活的操作方式（举个例子，C++没有线程优先级和亲和性的概念）
    - 为了提供对底层系统级线程API的访问，`std::thread`对象提供了`native_handle`的成员函数，而`std::future`（即`std::async`返回的东西）没有这种能力
  - 第二，需要且能够优化应用的线程使用，例如要开发一款已知执行概况的服务器软件，部署在有固定硬件特性的机器上，作为唯一的关键进程
  - 第三，需要实现C++并发API之外的线程技术，比如，C++实现中未支持的平台的线程池

### 36. Specify std::launch::async if asynchronicity is essential

- 当调用`std::async`执行函数时（或者其他可调用对象），通常希望异步执行函数，但是事实并不一定是你所想的那样，因为`std::async`是按照启动策略来执行的，有两种标准策略
  - `std::launch::async`启动策略意味着函数必须异步执行，即在不同的线程
  - `std::launch::deferred`启动策略意味着函数仅当在`std::async`返回的future上调用get或者wait时才执行，这表示函数推迟到存在这样的调用时才执行（注：异步与并发是两个不同概念，这里侧重于惰性求值）
    - 当get或wait被调用，函数会同步执行，即调用方被阻塞，直到函数运行结束，如果get和wait都没有被调用，函数将不会被执行（此处是简化说法，关键点不是要在其上调用get或wait的那个future，而是future引用的那个共享状态）

- 可能让人惊奇的是，`std::async`的默认启动策略不是上面中任意一个，而是求或在一起的`std::launch::async | std::launch::deferred`
  - 因此默认策略允许函数异步或者同步执行，这种灵活性允许`std::async`和标准库的线程管理组件承担线程创建和销毁的责任，避免资源超额，以及平衡负载
  - 但是，使用默认启动策略的`std::async`也有一些有趣的影响，假如给定一个线程t执行语句`auto fut = std::async(f);`
    - 无法预测f是否会与t并发运行，因为f可能被安排延迟运行
    - 无法预测f是否会在与某线程相异的另一线程上执行，这个某线程在fut上调用get或wait，如果对fut调用函数的线程是t，含义就是无法预测f是否在异于t的另一线程上执行
    - 无法预测f是否执行，因为不能确保在程序每条路径上，都会不会在fut上调用get或者wait

- 默认启动策略的调度灵活性也会带来一些问题
  - 首先是导致访问`thread_local`的不确定性，因为这意味着如果f读写了线程本地存储（thread-local storage，TLS），不可能预测到哪个线程的变量被访问
    - 因为f的TLS可能是为单独的线程建的，也可能是为在fut上调用get或者wait的线程建的
  - 其次是隐含了任务可能不会被执行的意思，会影响调用基于超时的wait的程序逻辑
    - 因为在一个延时的任务上调用`wait_for`或者`wait_until`会产生`std::launch::deferred`值，意味着，以下循环看似应该最终会终止，但可能实际上永远运行

    ```C++
    auto fut = std::async(f);           //异步运行f（理论上）
    // 有问题的设计
    while (fut.wait_for(100ms) !=       //循环，直到f完成运行时停止...
          std::future_status::ready)   //但是有可能永远不会发生！
    {
        …
    }
    // 修复后的设计（只需要检查与std::async对应的future是否被延迟执行即可，那样就会避免进入无限循环）
    if (fut.wait_for(0s) ==                 //如果task是deferred（被延迟）状态
        std::future_status::deferred)
    {
        …                                   //在fut上调用wait或get来异步调用f
    } else {                                //task没有deferred（被延迟）
        while (fut.wait_for(100ms) !=       //不可能无限循环（假设f完成）
              std::future_status::ready) {
            …                               //task没deferred（被延迟），也没ready（已准备）
                                            //做并行工作直到已准备
        }
        …                                   //fut是ready（已准备）状态
    }
    ```

- 这些各种考虑的结果就是，只要满足以下条件，`std::async`的默认启动策略就可以使用
  - 任务不需要和执行get或wait的线程并行执行
  - 读写哪个线程的`thread_local`变量没什么问题
  - 可以保证会在`std::async`返回的future上调用get或wait，或者该任务可能永远不会执行也可以接受
  - 使用`wait_for`或`wait_until`编码时考虑到了延迟状态
  - 但是如果上述条件任何一个都满足不了，你可能想要保证`std::async`会安排任务进行真正的异步执行，进行此操作的方法是调用时，将`std::launch::async`作为第一个实参传递

### 37. Make std::threads unjoinable on all paths

- 每个`std::thread`对象处于两个状态之一：可结合的（joinable）或者不可结合的（unjoinable）
  - 可结合状态的`std::thread`对应于正在运行或者可能要运行的异步执行线程
    - 比如，对应于一个阻塞的（blocked）或者等待调度的线程的`std::thread`是可结合的，对应于运行结束的线程的`std::thread`也可以认为是可结合的
  - 相应的，不可结合状态的`std::thread`则包括
    - 默认构造的`std::thread`对象，这种`std::thread`没有函数执行，因此没有对应到底层执行线程上
    - 已经被移动走的`std::thread`对象，移动的结果就是一个`std::thread`原来对应的执行线程现在对应于另一个`std::thread`
    - 已经被join的`std::thread`，在join之后，`std::thread`不再对应于已经运行完了的执行线程
    - 已经被detach的`std::thread` ，detach断开了`std::thread`对象与执行线程之间的连接

- `std::thread`的可结合性如此重要的原因之一就是当可结合的线程的析构函数被调用，程序执行会终止，因此必须要保证在代码执行的所有路径上保证thread最终是不可结合的
  - 你可能会想，为什么`std::thread`析构的行为是这样的，那是因为另外两种显而易见的方式更糟，考虑如下示例

    ```C++
    constexpr auto tenMillion = 10000000;           //constexpr见条款15

    bool doWork(std::function<bool(int)> filter,    //返回计算是否执行；
                int maxVal = tenMillion)            //std::function见条款2
    {
        std::vector<int> goodVals;                  //满足filter的值

        std::thread t([&filter, maxVal, &goodVals]  //填充goodVals
                      {
                          for (auto i = 0; i <= maxVal; ++i)
                              { if (filter(i)) goodVals.push_back(i); }
                      });

        auto nh = t.native_handle();                //使用t的原生句柄
        …                                           //来设置t的优先级

        if (conditionsAreSatisfied()) {
            t.join();                               //等t完成
            performComputation(goodVals);
            return true;                            //执行了计算
        }
        return false;                               //未执行计算
    }
    ```

    - 第一种，隐式join，这种情况下，`std::thread`的析构函数将等待其底层的异步执行线程完成
      - 这听起来是合理的，但是可能会导致难以追踪的异常表现。比如，如果`conditonAreStatisfied()`已经返回了false，`doWork`继续等待过滤器`filter`应用于所有值就很违反直觉
    - 第二种，隐式detach，这种情况下，`std::thread`析构函数会分离`std::thread`与其底层的线程，底层线程继续运行
      - 听起来比join的方式好，但是可能导致更严重的调试问题，比如在`doWork`中，`goodVals`是通过引用捕获的局部变量，它会被lambda修改，假定lambda异步执行时，`conditionsAreSatisfied()`返回false，这时`doWork`返回，同时局部变量（包括`goodVals`）被销毁，栈被弹出，并在`doWork`的调用点继续执行线程
      - 想象一下这会带来什么问题，`goodVals`已经被销毁，但是线程仍然在运行，它会继续修改`goodVals`，这会导致未定义行为
  - 每当你想在执行跳至块之外的每条路径执行某种操作，最通用的方式就是将该操作放入局部对象的析构函数中，这些对象称为RAII对象（Resource Acquisition Is Initialization objects），从RAII类中实例化

### 38. Be aware of varying thread handle destructor behavior

- 存储被调用者结果的位置被称为共享状态（shared state），共享状态通常是基于堆的对象，但是标准并未指定其类型、接口和实现，共享状态的存在非常重要，因为future的析构函数取决于与future关联的共享状态
  - 引用了共享状态（使用`std::async`启动的未延迟任务建立的那个）的最后一个future的析构函数会阻塞住，直到任务完成，本质上，这种future的析构函数对执行异步任务的线程执行了隐式的join
  - 其他所有future的析构函数简单地销毁future对象
    - 对于异步执行的任务，就像对底层的线程执行detach
    - 对于延迟任务来说如果这是最后一个future，意味着这个延迟任务永远不会执行了

- 这些规则听起来好复杂，我们真正要处理的是一个简单的“正常”行为以及一个单独的例外
  - 正常行为是future析构函数销毁future，那意味着不join也不detach，也不运行什么，只销毁future的数据成员
  - 正常行为的例外情况仅在某个future同时满足下列所有情况下才会出现，此时future的析构函数才会表现“异常”行为，就是在异步任务执行完之前阻塞住，这相当于对由于运行`std::async`创建出任务的线程隐式join
    - 它关联到由于调用`std::async`而创建出的共享状态
    - 任务的启动策略是`std::launch::async`，原因是运行时系统选择了该策略，或者在对`std::async`的调用中指定了该策略
    - 这个future是关联共享状态的最后一个future
      - 对于`std::future`，情况总是如此
      - 对于`std::shared_future`，如果还有其他的`std::shared_future`，与要被销毁的future引用相同的共享状态，则要被销毁的future遵循正常行为（即简单地销毁它的数据成员）

- future的API没办法确定是否future引用了一个`std::async`调用产生的共享状态，因此给定一个任意的future对象，无法判断会不会阻塞析构函数从而等待异步任务的完成
  - 当然，如果你有办法知道给定的future不满足上面条件的任意一条（比如由于程序逻辑造成的不满足），你就可以确定析构函数不会执行“异常”行为
  - 比如，只有通过`std::async`创建的共享状态才有资格执行“异常”行为，但是有其他创建共享状态的方式
    - 一种是使用`std::packaged_task`，一个`std::packaged_task`对象通过包覆（wrapping）方式准备一个函数（或者其他可调用对象）来异步执行，然后将其结果放入共享状态中，然后通过`std::packaged_task`的`get_future`函数可以获取有关该共享状态的future

### 39. Consider void futures for one-shot event communication

- 对于一次性事件通信考虑使用void的futures
  - 对于简单的事件通信，基于条件变量的设计需要一个多余的互斥锁，对检测和反应任务的相对进度有约束，并且需要反应任务来验证事件是否已发生
  - 基于flag的设计避免的上一条的问题，但是是基于轮询，而不是阻塞
  - 条件变量和flag可以组合使用，但是产生的通信机制很不自然
  - 使用`std::promise`和future的方案避开了这些问题，但是这个方法使用了堆内存存储共享状态，同时有只能使用一次通信的限制

### 40. Use std::atomic for concurrency, volatile for special memory

- 可怜的`volatile`本不应该出现在此处，因为它跟并发编程没有关系，但是在其他编程语言中（比如，Java和C#），`volatile`是有并发含义的，即使在C++中，有些编译器在实现时也将并发的某种含义加入到了`volatile`关键字中（但仅仅是在用那些编译器时），因此在此值得讨论下关于`volatile`关键字的含义以消除异议
  - `std::atomic`用于在不使用互斥锁情况下，来使变量被多个线程访问的情况，是用来编写并发程序的一个工具
  - `volatile`用在读取和写入不应被优化掉的内存上，意味着告诉编译器“不要对这块内存执行任何优化”，是用来处理特殊内存的一个工具

## CHAPTER 8 Tweaks

### 41. Consider pass by value for copyable parameters that are cheap to move and always copied

- 对于可拷贝，移动开销低，而且无条件被拷贝的形参，按值传递效率基本与按引用传递效率一致，而且易于实现，还生成更少的目标代码
  - 通过构造拷贝形参可能比通过赋值拷贝形参开销大的多
  - 按值传递会引起切片问题，所说不适合基类形参类型

### 42. Consider emplacement instead of insertion

- 置入（emplacement）函数可以完成插入函数的所有功能，并且有时效率更高，至少在理论上，不会更低效
  - 那为什么不在所有场合使用它们？因为，就像说的那样，只是“理论上”，但是实际上区别还是有的
  - 在当前标准库的实现下，有些场景，就像预期的那样，置入执行性能优于插入，但是，有些场景反而插入更快
  - 这种场景不容易描述，因为依赖于传递的实参的类型、使用的容器、置入或插入到容器中的位置、容器中类型的构造函数的异常安全性，和对于禁止重复值的容器（即`std::set`，`std::map`，`std::unordered_set`，`set::unordered_map`）要添加的值是否已经在容器中

- 当然这个结论不是很令人满意，所以你会很高兴听到还有一种启发式的方法来帮助你确定是否应该使用置入，如果下列条件都能满足，置入会优于插入
  - 值是通过构造函数添加到容器，而不是直接赋值，例如将值添加到`std::vector`末尾，一个先前没有对象存在的地方，新值必须通过构造函数添加到`std::vector`
    - 但是如果看下面这个例子，新值放到已经存在了对象的一个地方，那情况就完全不一样了
      - 对于这份代码，没有实现会在已经存在对象的位置`vs[0]`构造这个添加的`std::string`，而是通过移动赋值的方式添加到需要的位置，但是移动赋值需要一个源对象，所以这意味着一个临时对象要被创建，而置入优于插入的原因就是没有临时对象的创建和销毁，所以当通过赋值操作添加元素时，置入的优势消失殆尽

      ```C++
      std::vector<std::string> vs;        //跟之前一样
      …                                   //添加元素到vs
      vs.emplace(vs.begin(), "xyzzy");    //添加“xyzzy”到vs头部
      ```

  - 传递的实参类型与容器的初始化类型不同
    - 再次强调，置入优于插入通常基于以下事实：当传递的实参不是容器保存的类型时，接口不需要创建和销毁临时对象
    - 当将类型为`T`的对象添加到`container<T>`时，没有理由期望置入比插入运行的更快，因为不需要创建临时对象来满足插入的接口
  - 容器不拒绝重复项作为新值，这意味着容器要么允许添加重复值，要么你添加的元素大部分都是不重复的
    - 这样要求的原因是为了判断一个元素是否已经存在于容器中，置入实现通常会创建一个具有新值的节点，以便可以将该节点的值与现有容器中节点的值进行比较
    - 如果要添加的值不在容器中，则链接该节点，如果值已经存在，置入操作取消，创建的节点被销毁，意味着构造和析构时的开销被浪费了
    - 这样的节点更多的是为置入函数而创建，相比起为插入函数来说
