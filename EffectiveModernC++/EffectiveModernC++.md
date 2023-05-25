# Effective Modern C++

## Deducing Types

### 1. Understand template type deduction

- 以如下模板为例，T的类型推导不仅取决于expr的类型，也取决于ParamType的类型，这里有三种情况

  ```C++
  template<typename T>
  void f(ParamType param);

  f(expr); // 从expr中推导T和ParamType
  ```

  - ParamType是一个指针或引用，但不是通用引用，即`T& param`（关于通用引用请参见Item24。在这里你只需要知道它存在，而且不同于左值引用和右值引用）
    - 在这种情况下，如果expr的类型是一个引用，忽略引用部分，然后expr的类型与ParamType进行模式匹配来决定T
  - ParamType是一个通用引用，即`T&& param`
    - 如果expr是左值，T和ParamType都会被推导为左值引用。这非常不寻常，第一，这是模板类型推导中唯一一种T被推导为引用的情况。第二，虽然ParamType被声明为右值引用类型，但是最后推导的结果是左值引用。
    - 如果expr是右值，就使用正常的（也就是情景一）推导规则
  - ParamType既不是指针也不是引用，即`T param`
    - 当ParamType既不是指针也不是引用时，通过传值（pass-by-value）的方式处理，和之前一样，如果expr的类型是一个引用，忽略这个引用部分
    - 如果忽略expr的引用性（reference-ness）之后，expr是一个const，那就再忽略const。如果它是volatile，也忽略volatile（volatile对象不常见，它通常用于驱动程序的开发中。关于volatile的细节请参见Item40）
    - 但是此处有一个特殊情况，考虑`const char* const ptr = "Hello"`，此时ParamType被推导为`const char*`
      - 这种情况，ptr自身的值会被传给形参，根据类型推导的第三条规则，ptr自身的常量性constness将会被省略，所以param是`const char*`，也就是一个可变指针指向const字符串。在类型推导中，这个指针指向的数据的常量性constness将会被保留，但是当拷贝ptr来创造一个新指针param时，ptr自身的常量性constness将会被忽略。

- 另一个需要注意的问题是数组实参的类型推导，虽然数组和指针有时可以互换，但是两者并不相同
  - 对于`T param`，因为数组形参会视作指针形参，所以T被推导为指针，而不是数组
  - 但是对于`T& param`，虽然函数不能声明形参为真正的数组，但是可以接受指向数组的引用，例如对于`const char name[] = "J. P. Briggs";`，T被推导为`const char[13]`，形参（对这个数组的引用）的类型则为`const char (&)[13]`

- 最后一个细节是函数实参，不只是数组会退化为指针，函数类型也会退化为一个函数指针，我们对于数组类型推导的全部讨论都可以应用到函数类型推导和退化为函数指针上来
  - 指向函数的指针和指向函数的引用，实际上没有什么不同，但是如果你知道数组退化为指针，你也会知道函数退化为指针。

### 2. Understand auto type deduction

- auto类型推导通常和模板类型推导相同，但是auto类型推导假定花括号初始化代表std::initializer_list，而模板类型推导不这样做

- 在C++14中auto允许出现在函数返回值或者lambda函数形参中，但是它的工作机制是模板类型推导那一套方案，而不是auto类型推导

### 3. Understand decltype

- 通常，decltype会精确的告诉你你想要的结果，相比模板类型推导和auto类型推导，decltype只是简单的返回名字或者表达式的类型
  - decltype最主要的用途就是用于声明函数模板，而这个函数返回类型依赖于形参类型

- 但是对于T类型的不是单纯的变量名的左值表达式，decltype总是产出T的引用即`T&`，当使用`decltype(auto)`的时候一定要加倍的小心，在表达式中看起来无足轻重的细节将会影响到推导结果

  ```C++
  decltype(auto) f2()
  {
      int x = 0;
      return (x); //decltype((x))是int&，所以f2返回int&
  }
  ```

### 4. Know how to view deduced types

- 类型推断可以从IDE看出，从编译器报错看出，从Boost TypeIndex库的使用看出

- 这些工具可能既不准确也无帮助，所以理解C++类型推导规则才是最重要的

## auto

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

## Moving to Modern C++

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
  - 限域enum的枚举名仅在enum内可见。要转换为其它类型只能使用cast
  - 非限域/限域enum都支持底层类型说明语法，限域enum底层类型默认是int，非限域enum没有默认底层类型
  - 限域enum总是可以前置声明。非限域enum仅当指定它们的底层类型时才能前置声明

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


