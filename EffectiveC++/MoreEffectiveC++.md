# More Effective C++

## Basic

### 1. Distinguish between pointers and references

- 如果一个变量被用来指向一个对象，且需要具有指向其他对象的能力，但是也可能不指向任何对象，那么应该使用pointer，因为其可以被设为null

- 如果一个变量必须代表一个对象，而且绝不会改变指向其他对象，或者当实现一个操作符而其语法需求无法由pointer达成，那么应该使用reference

### 2. Prefer C++-style casts

- `static_cast`用于执行强迫隐式转换，拥有与C旧式转型几乎相同的威力与意义，以及相同的限制

- `const_cast`用于改变表达式中的常量性（constness）或易变性（volatileness）

- `dynamic_cast`用于继承体系中安全的向下转型或跨系转型动作，如果转型失败，当转型对象是指针时会返回null，当转型对象是reference时会抛出异常

- `reinterpret_cast`用于执行低级强制转型，转换结果与编译平台息息相关，所以不具备移植性

### 3. Never treat arrays polymorphically

- C++允许通过base class的pointers和references来操作“derived class objects所形成的数组”，但是这绝不值得沾沾自喜，因为它几乎绝不会如你所预期般的运作

- 假设有class Base，以及一个继承自base的class Derived，现有代码如下：

  ```C++
  void printAndDelete(ostream& s, Base array[], int size)
  {
    for (int i = 0; i < size; ++i) {
      s << array[i]; // 假设Base有operator<<可用
    }
    delete [] array; // 3. 此处呢？
  }

  Base b[10];
  ...
  print(cout, b, 10); // 1. 运行正常

  Derived d[10];
  ...
  print(cout, d, 10); // 2. 看起来运行正常，实际呢？
  ```

  - 编译器会毫无怨言地接受这段代码，但是后两处对数组的操作，其结果是不可预期的
    - `array[i]`是一个指针算术表达式，其含义是`*(array+i)`，那么`array`所指内存地址和`array+i`所指内存地址之间的差值是多少呢？答案是`i*sizeof(Base)`
    - 如果编译器拿到的`array`是由Derived对象所形成的数组，编译器就会被误导，其仍假设每一个元素的大小是Base的大小，通常Derived要比Base有更多的data members，所以编译器产生的指针算术表达式是错误的
    - 现在来看第三处的delete语句，当数组被删除时，每一个元素的析构函数都必须被调用，那么编译器可能会产出类似这样的代码：

      ```C++
      // 以*array中对象构造顺序的逆序来析构
      for (int i = size - 1; i >= 0; --i) {
        array[i].Base::~Base();
      }
      ```

      - 如果编译器产生类似的代码，毫无疑问是个错误的行为，C++语言规范中说，通过base class指针删除一个由derived class对象构成的数组，其结果未定义

- 总结来说，多态和指针算术绝不能混用，而数组对象几乎总是涉及到指针算术运算，所以数组和多态不要混用
  - 如果避免让一个具体类继承自另一个具体类，你就不太能够犯“以多态方式来处理数组”的错误，条款33有更多的讨论

### 4. Avoid gratuitous default constructors

- 在一个“完美的世界中”，凡是可以合理的从无到有生成对象的classes，都应该内含default constructors，而必须有某些外来信息才能生成对象的classes，则不应拥有default constructors

- 但是现实是，如果class缺乏default constructors，其运行可能在3种情况下出现问题
  - 第一个情况是在产生数组时，一般而言没有任何办法可以为数组中的对象指定构造函数自变量，所以几乎不可能产生一个由该对象构成的数组，例如`Widget ws[10];`或`Widget *ws = new Widget[10];`都是行不通的，由3个方法可以侧面解决这个问题
    - 第一个方法是使用non-heap数组，便能够在定义数组时提供必要的自变量，例如`Widget ws[10] = { Widget(1), Widget(2), Widget(3), ... };`，但是此方法无法用于heap数组
    - 第二个做法更一般化，是使用“指针数组”而非“对象数组”，但此方法有两个缺点，其一是必须将数组所指的所有对象删除，其二是内存使用会更大，因为每一个指针都要占用额外的内存空间，不过第二个确定可以利用“placement new”来避免

      ```C++
      // 分配足够的内存来容纳10个Class对象
      void *rawMemory = operator new[](10 * sizeof(Widget));
      // 数组指向这块内存
      Widget *ws = static_cast<Widget *>(rawMemory);
      // 用placement new来构造这10个Class对象
      for (int i = 0; i < 10; ++i) {
        new (&ws[i]) Widget(i); // placement new调用带参数的构造函数
      }
      
      ... // 使用ws数组

      // 以其构造顺序的逆序来析构
      for (int i = 9; i >= 0; --i) {
        ws[i].~Widget();
      }
      // 释放内存
      operator delete[](rawMemory);
      ```

      - 此方法的缺点是相当一部分程序员不熟悉，维护起来比较困难，而且需要在数组使用结束后，手动调用其析构函数，最后还要调用`operator delete[]`来释放内存，这都是很容易出错的地方
        - 你可能好奇如果采用`delete [] ws`来释放内存会发生什么，答案是不可预期，因为删除一个不是以`new` operator获得的指针，其结果是未定义的
  - 第二个缺点是，其将不适用于许多template-based container classes，因为在这些模板内几乎总是会产生一个以“模板类型参数”作为类型而架构起来的数组
    - 例如`template<class T> class Array { ... }`，Array的构造函数内可能包含`data = new T[size];`这样的数组产生代码，如果T没有default constructor，那么模板类也就无法使用了
    - 大多数情况下，如果谨慎设计template，可以消除对default constructors的需求，不幸的是许多设计者什么都有，独缺谨慎
  - 第三个考虑点和virtual base classes有关，virtual base classes如果缺乏default constructors，与之合作将是一种痛苦
    - 因为virtual base classes的构造函数自变量必须由派生层次最深的class提供，这就要求其所有的derived classes都必须知道且了解变量的意义，并提供构造函数所需的变量值

- 虽然有以上种种原因，但是仍然不建议添加无意义的default constructors，尽管这可能会对classes的使用带来某种限制，但是也带来了一种保证，即当你使用classes时，你可以预期该对象会被完全初始化，实现上亦富有效率

## Operators

### 5. Be wary of user-defined conversion functions

- C++中有两种函数允许编译器执行隐式转换
  - 第一种是单自变量构造函数，如此的构造函数可能声明拥有单一参数，如`class Name { public: Name(const string& s); };`，也可能声明拥有多个参数，但其他参数都有默认值，如`class Widget { public: Widget(int m = 0, int n = 1); };`
  - 第二种是隐式类型转换操作符，这是一个拥有奇怪名称的member function，需要在关键词`operator`后加一个类型名，且不能指定返回值类型，因为其返回值类型基本上已经体现在函数名称上了，如`class Rational { public: operator double() const; };`，这会将Rational对象转换为double类型

- 现在来考虑，为什么最好不要提供任何类型转换函数，根本问题在于，在你从未打算也未预期的情况下，此类函数可能会被调用，而其结果可能是不正确、不直观的程序行为，很难调试

- 先来考虑隐式类型转换操作符，因为其比较容易掌握
  - 假设有一个class Rational，你希望输出其内容`cout << Rational(1, 2);`，期望的结果是“1/2”，但是你忘记了为其提供`operator<<`，因此你或许认为该代码会执行失败
    - 但是很遗憾，编译器会想尽各种办法（包括找出一系列可接受的隐式类型转换），此时编译器发现只要调用`operator double()`，该动作便能成功，于是你会发现输出结果是一个浮点数
    - 上述问题虽然不至于造成灾难，却显示了隐式类型转换操作符的危险性，它们的出现可能导致错误（非预期）的函数被调用
  - 解决办法就是以功能对等的另一个函数取代类型转换操作符，如`double asDouble() const;`，如此的member function必须被明确调用，尽管这会带来些许不便，但却是值得的
    - 就像C++标准程序库的`string`类型，它提供了一个显式的`c_str()`函数，而不是隐式转换函数，巧合吗？我想不是

- 通过单自变量构造函数完成隐式转换则较难消除，而且其造成的问题在许多方面更难对付
  - 假设有一个针对数组结构而编写的class template

    ```C++
    template<class T> class Array {
    public:
      Array(int size); // 单自变量构造函数，可用于隐式转换
      Array(int lowBound, int highBound); // 允许指定索引值的范围
      T& operator[](int index);
      ...
    };

    // 假设有一个bool operator==(const Array<int>& lhs, const Array<int>& rhs)函数用来进行比较
    // 以及如下一段代码
    Array<int> a(10);
    Array<int> b(10);
    ...
    for (int i = 0; i < 10; ++i) {
      if (a == b[i]) { // 笔误，应该是a[i] == b[i]
        ...
      }
    }
    ```

    - 此时你一定无比希望编译器指出你的错误，但是结果却是它一声不吭，因为它看到的是`operator==`函数被调用，函数的两个参数类型分别是`Array<int>`和`int`，虽然没有这样的比较函数可被调用，但是只要使用Array的单自变量构造函数即可将`b[i]`转换为`Array<int>`类型，于是编译器这样做了
  - 虽然单自变量构造函数有这样的问题，但是却很难去除它，毕竟你可能真的需要一个这样的构造函数给用户使用，但是你又想阻止编译器不分青红皂白的进行隐式转换，幸运的是新版C++特性中的关键词`explicit`可以帮助你

### 6. Distinguish between prefix and postfix forms of increment and decrement operators

- C++中重载increment或decrement操作符的前置式和后置式如下

  ```C++
  class UPInt { // unlimited precision integer
  public:
    UPInt& operator++(); // 前置式++
    const UPInt operator++(int); // 后置式++
    UPInt& operator--(); // 前置式--
    const UPInt operator--(int); // 后置式--
    UPInt& operator+=(int); // +=操作，结合UPInt和int类型
    ...
  };
  ```

  - 由于重载函数是以其参数类型来区分彼此的，然而increment或decrement操作符的前置式和后置式都没有参数，为了对两者加以区分，只好让后置式有一个int类型的参数，并且在其被调用时，编译器默默为int类型的参数指定一个0值

- 让我们来关注一个更重要的区别，前置式操作返回一个reference，而后置式操作返回一个const对象，这是由其各自的操作意义所决定的

  ```C++
  // 前置式：累加然后取出（increment and fetch）
  UPInt& UPInt::operator++() {
    *this += 1;
    return *this;
  }
  // 后置式：取出然后累加（fetch and increment）
  const UPInt UPInt::operator++(int) {
    UPInt oldValue = *this;
    ++(*this);
    return oldValue;
  }
  ```

  - 后置式操作必须返回一个对象（代表旧值）的原因很清楚，但为什么是const对象呢？
    - 如果不加const，想象这样的操作`UPInt a; a++++;`，其将变成合法操作，但是我们并不欢迎这样的操作，理由有两个
    - 第一，它和内建类型的行为不一致，设计classes的一条无上宝典就是，一旦有疑惑，试看int行为如何并遵循之
    - 第二，即使能够两次施行后置式操作，其行为也非你所预期，因为第二次操作修改的对象是第一次操作返回的对象，而不是原对象

### 7. Never overload `&&`, `||`, or `,`

- C++对于“真假值表达式”采用所谓“骤死式（个人觉得应该叫短路式）”评估方式，意为一旦表达式的真假值确定，即使表达式中还有部分尚未验证，整个评估工作仍会结束

- C++允许用户重载`&&`和`||`操作符，但是这样做的后果是，从此“函数调用语义”将会取代“骤死式语义”，也就是说表达式`expr1 && expr2`将会被编译器视为`expr1.operator&&(expr2)`或`operator&&(expr1, expr2)`,而这两者的语义有两个重大区别
  - 第一，当函数动作被调用时，所有参数必须都被评估完成，换句话说，没有什么“骤死式语义”了
  - 第二，C++语言规范并未明确定义函数调用中各参数的评估顺序，所以无法知道`expr1`和`expr2`哪个会先被评估，这与“骤死式语义”形成明确的对比，后者总是自左向右评估表达式

- 另外一个不为人注意的操作符是逗号操作符`,`，其在for循环的更新区最为常见，如`for (int i = 0, j = size - 1; i < j; ++i, --j)`，在循环的最后一个成分中，i被累加而j被递减，这里很适合使用逗号操作符，因为for循环最后一个成分必须是个表达式
  - C++对于逗号操作符的内建行为规则是，逗号左侧会先被评估，然后右侧再进行评估，最后整个逗号表达式的结果以逗号右侧的值为代表，所以例子中编译器会先评估`++i`，然后是`--j`，而整个表达式的结果是`--j`的返回值
  - 至于你为什么需要知道这些，是因为如果你打算撰写自己的逗号操作符（个人感觉很疯狂，之前从没想过逗号也能重载），你必须模仿模仿这种行为，然而不幸的是，你无法做到这些必要的模仿，所以不要轻率的将逗号操作符重载

### 8. Understand the different meanings of new and delete

- 有时了解C++会让人觉得被刁难了，例如，请说明`new` operator和`operator new`之间的差异（注，本书中所说的`new` operator，即某些C++教程如C++ Primer所谓的`new` expression）
  - 当你写出`string *ps = new string("hello");`时，所使用就是`new` operator，这个操作符是语言内建的，不能被改变意义，总是做相同的事情，即分配内存，然后调用构造函数
  - 你能够改变的是用来容纳对象的那块内存的分配行为，`new` operator调用某个函数，执行必要的内存分配动作，你可以重写或重载那个函数，改变其行为，那个函数的名字叫做`operator new`，该函数的通常声明为`void* operator new(size_t size);`
  - 还记得条款4的内容吗，如果想要直接调用`operator new`，你可以像调用任何其他函数一样调用它`void *rawMemory = operator new(sizeof(string));`，但是它和`malloc`一样，唯一的任务就是分配内存，它不知道什么是构造函数

- 有时

## Exceptions

### 9. Use destructors to prevent resource leaks

### 10. Prevent resource leaks in constructors

### 11. Prevent exceptions from leaving destructors

### 12. Understand how throwing an exception differs from passing a parameter or calling a virtual function

### 13. Catch exceptions by reference

### 14. Use exception specifications judiciously

### 15. Understand the costs of exception handling

## Efficiency

### 16. Remember the 80-20 rule

### 17. Consider using lazy evaluation

### 18. Amortize the cost of exoected computations

### 19. Understand the origin of temporary objects

### 20. Facilitate the return value optimization

### 21. Overload to avoid implicit type conversions

### 22. Consider using `op=` instead of stand-alone `op`

### 23. Consider alternative libraries

### 24. Understand the costs of virtual functions, multiple inheritance, virtual base classes, and RTTI

## Techniques, Idioms, Patterns

### 25. Virtualizing constructors and non-member functions

### 26. Limiting the number of objects of a class

### 27. Requiring or prohibiting heap-based objects

### 28. Smart pointers

### 29. Reference counting

### 30. Proxy classes

### 31. Making functions virtual with respect to more than one object

## Miscellany

### 32. Program in the future tense

### 33. Make non-leaf classes abstract

### 34. Understand how to combine C++ and C in the same program

### 35. Familiarize yourself with the language standard
