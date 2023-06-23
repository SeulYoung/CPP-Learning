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
  - 不过，有一个解决方案，可以为 const volatile 参数声明复制构造函数，并将其标记为 delete，这样做可以防止隐式声明另一个复制构造函数
    - 此时就可以定义一个构造函数模板，对于非 volatile 类型，该构造函数将优先于 (已删除的) 复制构造函数，在模板构造函数中，可以使用 enable_if<> 进行约束

### 6.5 Using Concepts to Simplify enable_if<> Expressions
