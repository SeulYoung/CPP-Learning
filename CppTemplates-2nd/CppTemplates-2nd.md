# C++ Templates - The Complete Guide 2nd Edition

## Chapter 1 Function Templates

### 1.1 A First Look at Function Templates

- 模板“编译”分为两个阶段:
  - 若在定义时不进行实例化，则会检查模板代码的正确性，而忽略模板参数，这包括:
    - 语法错误，比如: 缺少分号
    - 使用未知名称的模板参数 (类型名、函数名……)
    - 检查 (不依赖于模板参数) 静态断言
  - 实例化时，再次检查模板代码，确保生成的代码有效，特别是所有依赖于模板参数的部分，都会进行重复检查

### 1.2 Template Argument Deduction

### 1.3 Multiple Template Parameters

### 1.4 Default Template Arguments

### 1.5 Overloading Function Templates

- 非模板函数可以与相同名称和相同类型的函数模板共存，其他相同的情况下，重载解析将优先使用非模板方式
  - 但是若模板可以生成匹配更好的函数实例，则选择模板
  - 也可以显式指定一个空的模板参数列表，这种语法表明只有模板才能解析调用，但所有模板参数都应该根据调用参数进行推导
  - 对推导的模板参数不会进行自动类型转换，但是会自动的对普通函数参数进行类型转换，因此此时调用会使用非模板函数

- 重载函数模板时，最好不要进行不必要的更改，可以将更改限制在参数的数量或显式地指定模板参数，
  - 否则，可能会出现意想不到的后果，示例如下

    ```C++
    1 #include <cstring>
    2
    3 // 任意类型的两个最大值(引用调用)
    4 template<typename T>
    5 T const& max (T const& a, T const& b)
    6 {
    7 return b < a ? a : b;
    8 }
    9
    10 // 最多两个C字符串(值调用)
    11 char const* max (char const* a, char const* b)
    12 {
    13 return std::strcmp(b,a) < 0 ? a : b;
    14 }
    15
    16 // 任何类型的最多三个值(引用调用)
    17 template<typename T>
    18 T const& max (T const& a, T const& b, T const& c)
    19 {
    20 return max (max(a,b), c); // 如果max(a,b)使用了按值调用就会出错
    21 }
    22
    23 int main ()
    24 {
    25 auto m1 = ::max(7, 42, 68); // OK
    26
    27 char const* s1 = "frederic";
    28 char const* s2 = "anica";
    29 char const* s3 = "lucas";
    30 auto m2 = ::max(s1, s2, s3); // 运行时错误（未定义的行为），因为对于 C 字符，max(a,b) 创建了一个新临时变量，并通过引用返回
    31 }
    ```

### 1.6 But, Shouldn’t We ...?

- 使用值，还是引用传递参数?
  - 为什么通常声明函数按值传递参数，而不是使用引用。一般来说，除了简单类型 (比如基本类型或 std::string_view)，因为不会创建副本，所以推荐使用引用传递
  - 然而，按值传递在下面的情况下会更好:
    - 语法简单
    - 编译器会进行很好的优化
    - 移动语义会使复制成本降低
    - 没有复制或移动操作
  - 此外，对于模板来说:
    - 模板可能同时用于简单类型和复杂类型，因此为复杂类型选择这种方法时，可能会对简单类型产生反效果
    - 作为调用者，可以通过引用来传递参数，可以使用 std::ref() 和 std::cref()
    - 虽然传递字符串字面值或原始数组可能会产生问题，但通过引用传递通常会有更多的问题
  - 这些将在第 7 章中详细讨论，目前，我们使用值传递参数 (除非某些功能只有在使用引用时才使用引用)

- 为什么不用内联?
  - 通常，函数模板不必使用内联声明，与普通的非内联函数不同，我们可以在头文件中定义非内联函数模板，并在多个翻译单元中包含该头文件
  -该规则的唯一例外是，对特定类型的模板进行完全特化，从而产生的代码不再是泛型 (定义了所有模板参数)

- 为什么不用 constexpr?
  - C++11 后，可以使用 constexpr 提供在编译时计算某些值的能力，对于很多模板来说，这很有意义
  - 第 8.2 节将讨论使用 constexpr 的例子，但是为了让我们的注意力集中在基本特性上，在讨论其他模板特性时，我们通常会跳过 constexpr

### 1.7 Summary

## Chapter 2 Class Templates

### 2.1 Implementation of Class Template Stack

### 2.2 Use of Class Template Stack

- 对于类模板，只有在使用成员函数时才实例化，这节省了时间和空间，并且只允许使用部分地类模板
  - 如果类模板具有静态成员，则对于使用类模板的每个类型实例，这些成员也会实例化一次

### 2.3 Partial Usage of Class Templates

### 2.4 Friends

- 但当试图声明友元函数并实现时，事情会变得更加复杂，这里有两种选择
  - 隐式声明一个新的函数模板，但要使用不同的模板参数

    ```C++
    1 template<typename T>
    2 class Stack {
    3 ...
    4 template<typename U>
    5 friend std::ostream& operator<< (std::ostream&, Stack<U> const&);
    6 };
    ```

  - 可以将 `Stack<T>` 的输出操作符转发声明为模板，这样就需要转发声明 `Stack<T>`，注意“函数名”操作符 `<<` 后面的 `<T>`，因此，需要将非成员函数模板的特化声明为友元，若没有 `<T>`，需要声明新的非模板函数

    ```C++
    1 template<typename T>
    2 class Stack;
    3 template<typename T>
    4 std::ostream& operator<< (std::ostream&, Stack<T> const&);

    1 template<typename T>
    2 class Stack {
    3 ...
    4 friend std::ostream& operator<< <T> (std::ostream&,
    5 Stack<T> const&);
    6 };
    ```

### 2.5 Specializations of Class Templates

- 可以为某些模板参数特化类模板，类似于函数模板的重载，特化类模板允许优化特定类型的实现，或者为类模板的实例化修复特定类型的错误行为
  - 但若特化类模板，则必须特化所有成员函数，虽然可以特化类模板的单个成员函数，但若这样做了，就不能再特化该特化成员所属的整个类模板实例

### 2.6 Partial Specialization

- 类模板可偏特化，可以为特定的情况提供特殊的实现，但有些模板参数仍需要由用户定义
  - 若多个偏特化都匹配调用，则声明有歧义

### 2.7 Default Class Template Arguments

### 2.8 Type Aliases

### 2.9 Class Template Argument Deduction

- C++17 前，必须将所有模板参数类型传递给类模板 (除非有默认值)。C++17 后，指定模板参数的约束放宽了,相反，若构造函数能够推导出所有模板参数 (没有默认值)，则可以不用显式定义模板参数
  - 与函数模板不同，类模板参数不能只推导部分参数 (通过显式地只指定部分模板参数)

### 2.10 Templatized Aggregates

### 2.11 Summary

- 类模板是在实现时保留一个或多个类型参数的类
  - 要使用类模板，需要将类型作为模板参数传递，并为这些类型，实例化 (并编译) 类模板
  - 对于类模板，只有调用的成员函数会实例化
  - 可以为某些类型特化类模板
  - 可以偏特化某些类型的类模板
  - C++17 后，可以从构造函数中自动推导出类模板的参数
  - 可以定义聚合类模板
  - 若声明为按值调用，则模板类型的调用参数会衰变
  - 模板只能在全局/命名空间作用域或类声明内部声明和定义

## Chapter 3 Nontype Template Parameters

### 3.1 Nontype Class Template Parameters

### 3.2 Nontype Function Template Parameters

### 3.3 Restrictions for Nontype Template Parameters

- 非类型模板参数有一些限制，只能是整型常量值 (包括枚举)，指向对象/函数/成员的指针，指向对象或函数的左值引用，或者 std::nullptr_t(nullptr 的类型)
  - 当向指针或引用传递模板参数时，对象不能是字符串字面值、临时对象或数据成员和其他子对象

- 避免无效的表达式
  - 若在表达式中使用 >，必须将整个表达式放入圆括号中，以便编译器确定 > 在哪里结束
    `C<42, sizeof(int) > 4> c; // ERROR: first > ends the template argument list`
    `C<42, (sizeof(int) > 4)> c; // OK`

### 3.4 Template Parameter Type auto

- 从 C++17 开始，可以定义一个非类型的模板参数来泛化地接受任何允许用于非类型参数的类型
  - 例如对于`template<typename T, auto Maxsize>`，通过使用占位符类型 auto，可以将 Maxsize 定义为尚未指定类型的值，可以是任何允许为非类型模板参数的类型

### 3.5 Summary

- 模板的模板参数可以是值，而非类型
  - 不能将浮点数或类类型对象作为非类型模板的参数。对于指向字符串字面量、临时对象和子对象的指针/引用，有一些限制
  - 使用 auto 可使模板具有泛型值的非类型模板参数

## Chapter 4 Variadic Templates

### 4.1 Variadic Templates

### 4.2 Fold Expressions

- 下表列出了可能的折叠表达式，其中 op 是二元操作符，pack 是一个参数包，init 是一个初始化器
  | 折叠表达式 | 展开 |
  | :--- | :--- |
  | ( ... op pack ) | ((( pack1 op pack2 ) op pack3 ) ... op packN ) |
  | ( pack op ... ) | ( pack1 op ( ... ( packN-1 op packN ))) |
  | ( init op ... op pack ) | ((( init op pack1 ) op pack2 ) ... op packN ) |
  | ( pack op ... op init ) | ( pack1 op ( ... ( packN op init ))) |

### 4.3 Application of Variadic Templates

### 4.4 Variadic Class Templates and Variadic Expressions

- 参数包还可以出现在其他地方，例如表达式、类模板、using 声明，甚至推导策略等
  - Variadic Expressions

    ```C++
    template<typename T1, typename... TN>
    constexpr bool isHomogeneous (T1, TN...)
    {
    return (std::is_same<T1,TN>::value && ...); // since C++17
    }
    ```

  - Variadic Indices

    ```C++
    template<std::size_t... Idx, typename C>
    void printIdx (C const& coll)
    {
    print(coll[Idx]...);
    }
    ```

  - Variadic Class Templates

    ```C++
    // type for arbitrary number of indices:
    template<std::size_t...>
    struct Indices {
    };
    // This is a first step towards meta-programming, which will be discussed in Section 8.1
    template<typename T, std::size_t... Idx>
    void printByIdx(T t, Indices<Idx...>)
    {
    print(std::get<Idx>(t)...);
    }

    std::array<std::string, 5> arr = {"Hello", "my", "new", "!", "World"};
    printByIdx(arr, Indices<0, 4, 3>());
    ```
  
  - Variadic Deduction Guides

    ```C++
    namespace std {
    template<typename T, typename... U> array(T, U...)
    -> array<enable_if_t<(is_same_v<T, U> && ...), T>,
    (1 + sizeof...(U))>;
    }
    ```
  
  - Variadic Base Classes and using

    ```C++
    template<typename... Bases>
    struct Overloader : Bases...
    {
    using Bases::operator()...; // OK since C++17
    };
    // we can define a class derived from a variadic number of base classes that brings in the operator() declarations from each of those base classes
    using CustomerOP = Overloader<CustomerHash,CustomerEq>;
    // we use this feature to derive CustomerOP from CustomerHash and CustomerEq and enable both implementations of operator() in the derived class
    ```

### 4.5 Summary

- 通过使用参数包，可以为任意数量、类型的模板参数定义模板
- 要处理参数，需要递归和/或匹配的非变参函数
- 操作符 sizeof... 可为参数包提供的参数数量
- 可变参数模板的一个应用是转发任意数量的类型参数
- 通过使用折叠表达式，可以对参数包中的所有参数使用相应的操作符

## Chapter 5 Tricky Basics

### 5.1 Keyword typename

- 关键字 typename 是在 C++ 标准化过程中引入的，目的是说明模板内的标识符是类型

### 5.2 Zero Initialization

- 对于基本类型，如 int、double 或指针类型，没有默认构造函数可以初始化，任何未初始化的局部变量都有一个未定义的值
  - 因此，可以显式调用内置类型的默认构造函数，该构造函数用 0 初始化内置类型 (bool 为 false，指针为 nullptr)，这种初始化方式称为值初始化，要么调用提供的构造函数，要么对对象进行零初始化

### 5.3 Using this->

- 对于具有依赖于模板参数的基类类模板，即使成员 x 被继承，使用名称 x 本身并不总是等同于this->x
  - 目前，建议始终对基类中声明的符号进行限定，这些符号在某种程度上依赖于模板参数 `this->` 或 `Base<T>::`

### 5.4 Templates for Raw Arrays and String Literals

- 将原始数组或字符串字面值传递给模板时，务必小心
  - 若模板参数声明为引用，则参数不会衰变，传入的”hello” 参数为 `char const[6]` 类型，若因类型不同而传递不同长度的原始数组或字符串参数，这可能会成为一个问题
  - 只有当按值传递参数时，类型才会衰变，因此字符串字面值转换为 `char const*`
  - 注意，由语言规则声明为数组 (带或不带长度) 的调用参数实际上具有指针类型

  ```C++
  1 #include "arrays.hpp"
  2
  3 template<typename T1, typename T2, typename T3>
  4 void foo(int a1[7], int a2[], // pointers by language rules
  5 int (&a3)[42], // reference to array of known bound
  6 int (&x0)[], // reference to array of unknown bound
  7 T1 x1, // passing by value decays
  8 T2& x2, T3&& x3) // passing by reference
  9 {
  10 MyClass<decltype(a1)>::print(); // uses MyClass<T*>
  11 MyClass<decltype(a2)>::print(); // uses MyClass<T*>
  12 MyClass<decltype(a3)>::print(); // uses MyClass<T(&)[SZ]>
  13 MyClass<decltype(x0)>::print(); // uses MyClass<T(&)[]>
  14 MyClass<decltype(x1)>::print(); // uses MyClass<T*>
  15 MyClass<decltype(x2)>::print(); // uses MyClass<T(&)[]>
  16 MyClass<decltype(x3)>::print(); // uses MyClass<T(&)[]>
  17 }
  18
  19 int main()
  20 {
  21 int a[42];
  22 MyClass<decltype(a)>::print(); // uses MyClass<T[SZ]>
  23
  24 extern int x[]; // forward declare array
  25 MyClass<decltype(x)>::print(); // uses MyClass<T[]>
  26 foo(a, a, a, x, x, x, x);
  27
  28 }
  29
  30 int x[] = {0, 8, 15}; // define forward-declared array
  ```

### 5.5 Member Templates

- 类成员可以是模板，对于嵌套类和成员函数都可以是模板，成员函数模板允许全特化，但不能偏特化
  - 这种能力是有应用和优势的，例如有 `Stack<>` 类模板，其默认赋值操作符要求赋值操作符的两端具有相同的类型，但不能将其他类型的元素赋值给堆栈，即使对定义的元素类型有隐式的类型转换
  - 然而通过将赋值操作符定义为模板，可以使用定义了适当类型转换的元素对堆栈赋值，示例如下

    ```C++
    1 template<typename T, typename Cont = std::deque<T>>
    2 class Stack {
    3 private:
    4 Cont elems; // elements
    5
    6 public:
    7 void push(T const&); // push element
    8 void pop(); // pop element
    9 T const& top() const; // return top element
    10 bool empty() const { // return whether the stack is empty
    11 return elems.empty();
    12 }
    13
    14 // assign stack of elements of type T2
    15 template<typename T2, typename Cont2>
    16 Stack& operator= (Stack<T2,Cont2> const&);
    17 // to get access to private members of Stack<T2> for any type T2:
    18 template<typename, typename> friend class Stack;
    19 };
    ```

- 有一类特殊成员函数的模板，即模板构造函数或模板赋值操作符
  - 但是要注意即使定义了对应的模板，也不能取代预定义构造函数或赋值操作符，且成员模板不作为复制或移动对象的特殊成员函数
  - 上述例子中，对于相同类型的堆栈赋值，仍然使用默认赋值操作符，这利弊皆有
    - 模板构造函数或赋值操作符，可能比预定义的复制/移动构造函数或赋值操作符更匹配，尽管只提供了用于初始化其他类型的模板版本，参见 6.2 节
    - 要对复制/移动构造函数进行“模板化”以约束其存在是非常困难的，参见 6.4 节

- 有时，在调用成员模板时，需要显式限定模板参数，这种情况下，必须使用 template 关键字来确保 < 是模板参数列表的开头
  - .template 表示法 (以及类似的表示法，如->template 和::template) 应该只在模板内部使用，只有当它们遵循依赖于模板的某些参数时才应使用，示例如下，由于句点之前的构造依赖于模板参数，即参数 bs 依赖于模板参数 N，因此必须使用 .template 表示法

  ```C++
  1 template<unsigned long N>
  2 void printBitset (std::bitset<N> const& bs) {
  3 std::cout << bs.template to_string<char, std::char_traits<char>,
  4 std::allocator<char>>();
  5 }
  ```

### 5.6 Variable Templates

- C++17 后，标准库使用变量模板技术为标准库中产生 (布尔) 值的所有类型特征定义快捷方式，例如

  ```C++
  1 namespace std {
  2 template<typename T> constexpr bool is_const_v = is_const<T>::value;
  3 }
  ```

### 5.7 Template Template Parameters

- 模板模板参数允许模板参数本身是类模板，同样以堆栈类模板作为示例
  - 通过使用模板模板参数，可以通过指定容器的类型来声明 Stack 类模板，而无需重新指定容器元素的类型`Stack<int, std::vector> vStack;`

  ```C++
  1 template<typename T,
  2 template<typename Elem> class Cont = std::deque>
  3 class Stack {
  4 private:
  5 Cont<T> elems; // elements
  6
  7 public:
  8 void push(T const&); // push element
  9 void pop(); // pop element
  10 T const& top() const; // return top element
  11 bool empty() const { // return whether the stack is empty
  12 return elems.empty();
  13 }
  14 ...
  15 };
  ```

### 5.8 Summary

- 要访问依赖于模板参数的类型名称，必须使用 typename 对名称进行限定
- 要访问依赖于模板参数的基类成员，必须通过 this-> 或类名访问
- 嵌套类和成员函数也可以是模板，一种应用是能够实现具有内部类型转换的泛型操作
- 构造函数或赋值操作符的模板版本，不能替换预定义的构造函数或赋值操作符
- 通过使用带大括号的初始化或显式调用默认构造函数，即使是内置类型实例化，也可以确保使用默认值初始化模板的变量和成员
- 可以为原始数组提供特化的模板，这些模板也可以应用于字符串字面量
- 当传递原始数组或字符串字面量，且参数不是引用时，类型在推导过程中会衰变 (执行数组到指针的转换)
- 可以使用类模板作为模板参数，或作为模板模板参数，但模板模板参数必须精确匹配

## Chapter 6 Move Semantics and enable_if<>

### 6.1 Perfect Forwarding

- 在使用函数接受右值引用时，一个要注意的问题是，移动语义无法传递
  - 移动语义无法自动传递是有原因的，若不是这样，就会在函数中第一次使用可移动对象时丢失其内部值，因此需要显示的再次使用 std::move()

- 注意区分对于特定类型的右值引用`X&&`和对于模板参数的转发引用`T&&`（也称为通用引用）
  - 注意，T 必须是模板参数的名称，仅依赖模板参数是不够的，对于模板参数 T，像 `typename T::iterator&&` 这样的声明只是一个右值引用，而不是转发引用

### 6.2 Special Member Function Templates

- 成员函数模板也可以用作特殊的成员函数，包括用作构造函数，但这可能会导致意想不到的结果
  - 例如根据 C++ 的重载解析规则，对于一个非常量左值 Type t，成员模板要比 (通常是预定义的) 复制构造函数匹配的更好，因为对于复制构造函数，需要转换为 const
  - 另一个问题是，对于派生类的对象，成员模板仍然是更好的匹配，会劫持普通成员函数的调用

### 6.3 Disable Templates with enable_if<>

- 从 C++11 开始，标准库提供了辅助模板 std::enable_if<>，以在特定的编译时条件下忽略函数模板，std::enable_if<> 是一种类型特征，计算作为其 (第一个) 模板参数传递的给定编译时表达式
  - 若表达式结果为 true，其类型成员 `type` 将作为一个实际的类型
    - 若没有传递第二个模板参数，则该类型为 void
    - 否则，该类型就是第二个模板参数类型
  - 若表达式结果为 false，则没有定义类型成员 `type`。由于名为 SFINAE 的模板特性 (替换失败不是错误)参见 8.4 节，这将忽略使用 enable_if 表达式的函数模板

### 6.4 Using enable_if<>

- 可以使用 enable_if<> 来解决在 6.2 节中引入的构造函数模板的问题
  - 常用的是`std::is_convertible<FROM,TO>`或`std::is_constructible<T, Args...>`

- 禁用特殊成员函数的 tricky solution
  - 注意，通常不能使用 enable_if<> 来禁用预定义的复制/移动构造函数和/或赋值操作符，原因是成员函数模板永远不会算作特殊成员函数，并且在需要复制构造函数等情况下会忽略
  - 不过，有一个解决方案，可以用 const volatile 为参数声明复制构造函数，并将其标记为 delete，这样做可以防止隐式声明另一个复制构造函数
    - 此时就可以定义一个构造函数模板，对于非 volatile 类型，该构造函数将优先于 (已删除的) 复制构造函数，在模板构造函数中，可以使用 enable_if<> 进行约束

### 6.5 Using Concepts to Simplify enable_if<> Expressions

### 6.6 Summary

- 模板中，通过将参数声明为转发引用并在转发调用中使用 std::forward<>()，就可以“完美”地转发参数
- 使用完美转发成员函数模板时，可能会比预定义用于复制或移动对象的特殊成员函数更匹配，这可能会导致意想不到的问题
- 使用 std::enable_if<>，可以在编译时条件为 false 时禁用函数模板
- 通过 std::enable_if<>，可以避免单个参数的构造函数模板或赋值操作符模板比隐式生成的特殊成员函数更加匹配的问题
- 可以通过删除预定义的参数为 const volatile 的特殊成员函数，实现对特殊成员函数的模板化 (并应用 enable_if<>)

## Chapter 7 By Value or by Reference?

### 7.1 Passing by Value

- 按值传递参数时，原则上必须复制每个参数，每个参数都成为所传递实参的副本，对于类，作为副本创建的对象通常由复制构造函数初始化，因此调用复制构造函数的代价可能会很高
  - 然而，即使在按值传递参数时，也有方法来避免复制，编译器可能会优化复制对象的复制操作，并且通过移动语义，对复杂对象的操作也可以变得廉价，考虑以下代码
    - 第二次和第三次调用中，当直接调用函数模板来获取 prvalues，编译器通常会优化传递参数，这样就不会调用复制构造函数了
      - C++17 起，这种优化是必需的，C++17 前，不能优化复制的编译器，至少需要使用移动语义，这通常会使复制成本降低

    ```C++
    1 std::string returnString();
    2 std::string s = "hi";
    3 printV(s); // copy constructor
    4 printV(std::string("hi")); // copying usually optimized away (if not, move constructor)
    5 printV(returnString()); // copying usually optimized away (if not, move constructor)
    6 printV(std::move(s)); // move constructor
    ```

### 7.2 Passing by Reference

- 让我们来讨论一下引用传递的不同风格，所有情况下，引用都不会创建副本 (因为参数只引用传递的实参)，另外，引用传递参数永不衰变，然而，有些引用传递却也是不可行的
  - 底层实现中，通过引用传递参数是通过传递参数地址实现的，地址编码紧凑，将地址从调用方传递给被调用方的效率很高
  - 然而，在编译代码时，传递地址会给编译器带来不确定性，因为理论上，可以对该地址“可达”的值进行修改，因此，编译器必须假设其缓存的值 (通常在机器寄存器中) 可能在调用之后都无效，重新加载这些值的成本可能非常高
  - 你可能会认为我们传递常量引用时，难道编译器不能推断出这样的数值其实不会发生更改的吗，不幸的是，情况并非如此，因为调用者可以通过自己的非常量引用修改引用的对象
  - 内联可以减缓这种情况，如果编译器可以内联展开调用，就可以将调用和被调用放在一起进行推断，并且在许多情况下“看到”这个地址除了传递底层值外，没有其他用途

- 使用模板时一个棘手的问题是，当传递非常量引用时，若传递 const 参数，可能导致 arg 变成一个常量引用的声明，这意味着将允许传递右值，但这里其实需要的可能是左值
  - 这种情况下，修改函数模板中传递的参数是错误的，当函数完全实例化时 (这可能发生在编译的后期)，修改该值的尝试都将触发错误

  ```C++
  1 template<typename T>
  2 void outR (T& arg) {
  3 ...
  4 }

  1 std::string returnString();
  2 std::string s = "hi";
  3 outR(s); // OK: T deduced as std::string, arg is std::string&
  4 outR(std::string("hi")); // ERROR: not allowed to pass a temporary (prvalue)
  5 outR(returnString()); // ERROR: not allowed to pass a temporary (prvalue)
  6 outR(std::move(s)); // ERROR: not allowed to pass an xvalue

  1 std::string const c = "hi";
  2 outR(c); // OK: T deduced as std::string const
  3 outR(returnConstString()); // OK: same if returnConstString() returns const string
  4 outR(std::move(c)); // OK: T deduced as std::string const
  5 outR("hi"); // OK: T deduced as char const[3]
  ```

## 7.3 Using std::ref() and std::cref()

- C++11 起，可以让调用者决定函数模板参数是通过值传递，还是通过引用传递
  - 当模板声明为按值接受参数时，调用者可以使用 std::cref() 和 std::ref()，通过引用传递参数
  - 注意 std::cref() 不会改变模板中参数的处理，只是使用了一个技巧，用一个行为像是引用的对象来包装传递的参数
  - 事实上，这会创建了一个 `std::reference_wrapper<>` 类型的对象，引用原始参数，并按值传递了这个对象

### 7.4 Dealing with String Literals and Raw Arrays

### 7.5 Dealing with Return Values

- 众所周知，有时返回引用可能是麻烦的来源，因为引用的东西超出了控制范围，因此，有时你可能希望确保函数模板按值返回结果
  - 但是使用模板参数 T 并不能保证它不是引用，因为 T 有时可能会隐式推导为引用，即使 T 是由按值调用推导而来的模板参数，当显式指定模板参数为引用时，也可能成为引用类型

    ```C++
    1 template<typename T>
    2 T retR(T&& p) // p is a forwarding reference
    3 {
    4 return T{...}; // OOPS: returns by reference when called for lvalues
    5 }

    1 template<typename T>
    2 T retV(T p) // Note: T might become a reference
    3 {
    4 return T{...}; // OOPS: returns a reference if T is a reference
    5 }
    6
    7 int x;
    8 retV<int&>(x); // retT() instantiated for T as int&
    ```

- 安全起见，这里有两个选择
  - 使用类型特征 std::remove_reference<> 将类型 T 转换为非引用
  - 通过声明返回类型 auto 来让编译器推断返回值的类型 (C++14 起)，因为 auto 总会衰变

### 7.6 Recommended Template Parameter Declarations

- 正如前面所述，有不同的方式声明依赖于模板参数的参数:
  - 通过值传递参数:这种方法很简单，其衰变字符串字面值和数组，但不能为大型对象提供最佳性能，调用者可以通过使用 std::cref() 和 std::ref() 进行引用传递
  - 通过引用传递参数，这种方法通常可以为大型对象提供更好的性能，特别是在传递以下情况的参数时
    - 针对已有对象 (lvalue) 传递左值引用
    - 针对临时对象 (prvalue) 或标记为可移动 (xvalue) 的对象传递右值引用
    - 或者针对以上两者实现转发引用
    - 注意这些情况下，参数都不会衰变，所以在传递字符串字面值和其他数组时，可能需要特别注意，而对于完美转发引用，还必须注意此时模板参数可能隐式推导出引用类型

### 7.7 Summary

- 测试模板时，可以使用不同长度的字符串字面值
- 通过值传递的模板参数会衰变，而通过引用传递的模板参数不会衰变
- std::decay<> 允许在引用传递的模板中衰变参数
- 某些情况下，函数模板声明参数通过值传递时，允许通过 std::cref() 和 std::ref() 传递参数的引用
- 按值传递模板参数很简单，但可能无法获得最佳性能
- 按值传递参数给函数模板，除非有很好的理由不这样做
- 确保返回值通常按值传递 (注意模板参数不能直接被用来指定返回类型)
- 有时需要测量性能，不要依赖直觉，因为直觉很可能是错的

## Chapter 8 Compile-Time Programming

### 8.1 Template Metaprogramming

### 8.2 Computing with constexpr

### 8.3 Execution Path Selection with Partial Specialization

- 在编译时使用偏特化可以在不同实现之间进行选择
- 但是因为函数模板不支持偏特化，所以必须使用其他机制根据某些约束来更改函数实现，可供的选择包括
  - 带有静态函数的类
  - std::enable_if，在第 6.3 节中介绍
  - SFINAE 特性
  - 编译时 if 特性，该特性从 C++17 引入，将在第 8.5 节中介绍

### 8.4 SFINAE (Substitution Failure Is Not An Error)

- C++ 中，以各种参数类型重载的函数很常见，因此，当编译器看到对重载函数的调用时，必须考虑每个候选函数，评估调用参数，并选择最匹配的候选函数，当候选集包括函数模板的情况下，编译器首先必须确定为该候选对象使用哪些模板参数，然后在函数参数列表及其返回类型中替换这些参数，然后评估匹配程度
  - 但替换过程可能会遇到问题，可能产生毫无意义的构造，语言规则并不认为这种无意义的替换会导致错误，而具有这种问题的候选则会直接忽略
  - 这里描述的替换过程不同于按需实例化过程 (见 2.2 节)，即使是不需要实例化，也可以进行替换 (编译器评估是否需要)，其会直接替换函数声明 (但不是函数体) 中的内容

- 对于某些条件，找出并制定正确的表达式来 SFINAE 出函数模板并不容易
  - 例如，想确保函数模板 len() 对于具有 size_type 成员但没有 size() 成员函数的参数类型就会忽略，因为函数声明中没有对 size() 成员函数的要求，最终会在实例化时出错

    ```C++
    1 template<typename T>
    2 typename T::size_type len (T const& t)
    3 {
    4 return t.size();
    5 }
    6
    7 std::allocator<int> x;
    8 std::cout << len(x) << ’\n’; // ERROR: len() selected, but x has no size()
    ```
  
  - 有一种常见的模式可以用来处理这种情况
    - 使用尾部返回类型语法指定返回类型 (前面使用 auto，在末尾返回类型之前使用->)
    - 使用 decltype 和逗号操作符定义返回类型
    - 给出所有必须有效的表达式，并以逗号操作符分隔 (转换为 void 以防重载逗号操作符)
    - 在逗号操作符的末尾定义一个实际返回类型的对象

    ```C++
    1 template<typename T>
    2 auto len (T const& t) -> decltype( (void)(t.size()), T::size_type() )
    3 {
    4 return t.size();
    5 }
    ```

### 8.5 Compile-Time if

- 使用 if constexpr(…) 语法，编译器使用编译时表达式来决定是应用 then 部分，还是 else 部分
  - 不满足条件的代码将会变成废弃语句 (discarded statement)，意味着代码没有实例化
  - 代码没有实例化意味着只执行第一个翻译阶段 (the definition time)，检查语法的正确性和不依赖于模板参数的名称 (见 1.1.3 节)

  ```C++
  1 template<typename T>
  2 void foo(T t)
  3 {
  4 if constexpr(std::is_integral_v<T>) {
  5 if (t > 0) {
  6 foo(t-1); // OK
  7 }
  8 }
  9 else {
  10 undeclared(t); // error if not declared and not discarded (i.e. T is not integral)
  11 undeclared(); // error if not declared (even if discarded)
  12 static_assert(false, "no integral"); // always asserts (even if discarded)
  13 static_assert(!std::is_integral_v<T>, "no integral"); // OK
  14 }
  15 }
  ```

### 8.6 Summary

- 模板提供了在编译时进行计算的能力 (使用递归进行迭代，使用偏特化或三元操作符进行选择)
- 使用 constexpr 函数，用可在编译时上下文中调用的“普通函数”可以取代大多数编译时计算
- 使用偏特化，可以根据特定的编译时约束，在类模板的不同实现之间进行选择
- 模板只在需要且函数模板声明中的替换不会导致代码无效时才会被使用，这个原则称为 SFINAE
- SFINAE 可以用来提供只针对某些类型和/或约束的函数模板
- C++17 起，编译时 if 允许根据编译时条件 (甚至在模板外部) 启用或丢弃语句

## Chapter 9 Using Templates in Practice

### 9.1 The Inclusion Model

- 


## Appendix B Value Categories

### B.1 Traditional Lvalues and Rvalues

- C++中左值和右值最早是来源于C，但是在C++中其含义已经发生了变化
  - C++中术语左值现在有时称为本地化值，注意字符串字面值也是 (不可修改的) 左值
  - 而右值是纯粹的数学值 (如 7 或字符’a’)，不一定有相关的存储，它们的存在就为了计算，当它们被使用之后就不能被再次引用，除了字符串字面量之外的字面值都是右值

### B.2 Value Categories Since C++11

- 当 C++11 中引入右值引用以支持移动语义时，将表达式划分为左值和右值的方法已不足以描述 C++11 的所有语言行为，注意所有表达式仍然是左值或右值，但右值类别现在会进一步细分

  ```mermaid
  graph TB
    expression --> glvalue --> lvalue;
    expression --> rvalue --> prvalue;
    glvalue --> xvalue;
    rvalue --> xvalue;
  ```

- 上图中 C++11 的分类在 C++17 中仍然有效，不过分类重新表述为如下
  - glvalue (generalized lvalue)：是一个表达式，其计算值决定了对象、位域或函数 (即具有存储空间的实体，除位域外，glvalue 总是生成具有实际地址的实体) 的标识
  - prvalue (pure rvalue)：是一个表达式，其求值是初始化一个对象或位域，或计算操作符的操作数的值
  - xvalue (expiring value)：是一个 glvalue，指定一个对象或位域，其资源可以重用，通常是因为其即将” 过期”
  - lvalue：不是 xvalue 的 glvalue
  - rvalue：prvalue 或 xvalue 的表达式

- 需要强调的是，glvalue、prvalue、xvalue 等都是表达式，而不是值或实体
  - 例如，变量不是左值，尽管表示该变量的表达式是左值

- 我们知道，左值通常要进行到右值的转换 (如果使用 C++11 的值分类，此处为 glvalue 到 prvalue 的转换会更准确，但传统术语仍然更为常见) ，因为值是初始化对象的表达式类型
  - C++17 中，此处的转换有一个对偶形式被称为 temporary materialization (也可以称为 prvalue-to-xvalue 的转换)
    - 此处的含义是，如果一个 prvalue 有效的出现在了一个期望 glvalue 的地方，则一定会创建一个临时对象，该对象是由这个 prvalue 初始化的，随后这个原本的 prvalue 就会被 xvalue (也就是这个临时对象) 替换
    - 例如`int f(int const&); int r = f(3);`，此处 f() 是引用参数，所以需要一个 glvalue，但是表达式 3 是一个 prvalue，因此 temporary materialization 规则开始发挥作用，表达式 3 “转换”为一个 xvalue，该 xvalue 就是一个用值 3 初始化的临时对象
  - 在以下情况下一个临时对象会被创造并用 prvalue 进行初始化:
    - prvalue 绑定到引用
    - 访问类 prvalue 的成员
    - 数组 prvalue 的下标被访问
    - 数组 prvalue 转换为指向其第一个元素的指针 (数组衰变)
    - prvalue 出现在一个大括号的初始化列表中，例如对于某些类型 X，其初始化 `std::initializer_list<X>` 类型的对象
    - 将 sizeof 或 typeid 操作符应用于 prvalue
    - prvalue 是 `expr;` 形式的语句中的顶层表达式，或者将表达式转换为 void

- C++17 中，由 prvalue 初始化的对象由上下文决定，因此临时对象只在真正需要时才创建
  - C++17前，prvalue (特别是类) 总是隐含一个临时变量，虽然这些临时变量的副本可以在之后有选择地被省略，但是编译器仍然需要强制执行复制操作的大多数语义约束 (需要复制构造函数可调用)
  - 下面的例子展示了 C++17 修改规则的结果
    - C++17 之前，`prvalue N{}` 生成了一个类型为 `N` 的临时变量，但是允许编译器删除该临时变量的复制和移动 (实际上它们总是这样做的)
      - 这意味着调用 `make_N()` 的临时结果可以直接在 `n` 的存储中构造，不需要进行复制或移动操作，但 C++17 前的编译器仍需要检查是否可以进行复制或移动操作
      - C++17 中，prvalue `N` 本身不会产生一个临时变量，而是会初始化一个由上下文决定的对象
        - 在我们的例子中，这个对象就是 `n`，不需要考虑复制或移动操作 (这不是优化，而是C++语言保证)，因此此代码是有效的 C++17 代码

    ```C++
    1 class N {
    2 public:
    3 N();
    4 N(N const&) = delete; // this class is neither copyable ...
    5 N(N&&) = delete; // ... nor movable
    6 };
    7
    8 N make_N() {
    9 return N{}; // Always creates a conceptual temporary prior to C++17.
    10 } // In C++17, no temporary is created at this point.
    11
    12 auto n = make_N(); // ERROR prior to C++17 because the prvalue needs a conceptual copy.
    13 // OK since C++17, because n is initialized directly from the prvalue.
    ```

### B.3 Checking Value Categories with decltype

- 使用关键字 decltype，可以检查 C++ 表达式的值类别，对于任意表达式 x，`decltype((x))`(注意双括号) 会产生如下结果
  - 如果 x 为 prvalue，则结果为 x 的 type
  - 如果 x 为 lvalue，则结果为 x 的 type&
  - 如果 x 为 xvalue，则结果为 x 的 type&&

- `decltype((x))` 中需要双括号，以避免在表达式 x 确实是一个命名实体的情况下，产生命名实体的声明类型 (其他情况下，括号不起作用)
  - 例如若表达式 x 只是一个命名为 v 的变量，那么不带圆括号的构造就变成了 decltype(v)，其结果将是变量 v 的类型，而不是反映指向该变量的表达式 x 的值类别的类型

### B.4 Reference Types

- C++ 中的引用类型 (如 int&) 以两种重要的方式与值类别交互
  - 第一个是引用可能会限制可以绑定到该引用的表达式的值类别
    - 如 int& 类型的非 const 左值引用，只能用 int 类型的左值表达式初始化
    - 类似地，int&& 类型的右值引用，只能用 int 类型的右值表达式初始化
  - 第二种值类别与引用交互的方式是函数的返回类型，使用引用类型作为返回值的类型会影响对该函数调用的值类别
    - 对返回类型为左值引用函数的调用将产生左值
    - 对返回类型是对象右值引用的函数的调用将产生 xvalue (注意对函数类型的右值引用总是只会产生左值)
    - 对返回非引用类型的函数的调用会产生 prvalue
  - 下面的示例中，将演示引用类型和值类别之间的交互

    ```C++
    1 int& lvalue();
    2 int&& xvalue();
    3 int prvalue();

    1 int& lref1 = lvalue(); // OK: lvalue reference can bind to an lvalue
    2 int& lref3 = prvalue(); // ERROR: lvalue reference cannot bind to a prvalue
    3 int& lref2 = xvalue(); // ERROR: lvalue reference cannot bind to an xvalue
    4
    5 int&& rref1 = lvalue(); // ERROR: rvalue reference cannot bind to an lvalue
    6 int&& rref2 = prvalue(); // OK: rvalue reference can bind to a prvalue
    7 int&& rref3 = xvalue(); // OK: rvalue reference can bind to an xrvalue
    ```
