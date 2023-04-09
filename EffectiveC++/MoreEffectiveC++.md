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

- 有时你会有一些分配好的原始内存，需要在上面构建对象，此时需要使用特殊版本的`operator new`，称为placement new，例如`Widget *w = new (buffer) Widget();`，此对象将被构造在buffer所指的内存上，当程序运行于shared memory或memory-mapped I/O时，这种做法很有用

- 现在花几分钟想想并总结一下，两个术语虽然表面上令人迷惑，概念上却十分易懂
  - 如果你希望将对象产生于heap，请使用`new` operator，它不但会分配内存，还会为该对象调用构造函数
  - 如果你只打算分配内存，请使用`operator new`，那就没有任何构造函数会被调用
  - 如果你打算在heap objects产生时自己决定内存分配方式，请写一个自己的`operator new`，并使用`new` operator，它将自动调用你所写的`operator new`
  - 如果你打算在已分配的内存中构造对象，请使用placement new

- 为了避免资源泄露，每一个动态分配行为都必须匹配一个相应但相反的释放动作，函数`operator delete`对于内建的`delete` operator，就好像`operator new`对于`new` operator一样
  - 因此如果你只打算处理原始、未设初值的内存，应该完全回避`new` operator和`delete` operator，而是使用`operator new`分配内存并以`operator delete`释放内存
  - 内存释放动作是由函数`operator delete`执行，通常其声明为`void operator delete(void *rawMemory);`
  - 如果你使用placement new在某块内存中产生对象，你应该避免对那块内存使用`delete` operator，因为`delete` operator会调用`operator delete`来释放内存，但是该内存内含的对象最初并非是由`operator new`分配而来的，毕竟placement new只是返回它所接收的指针而已，谁知道那个指针是从何而来的
    - 所以为了抵消该对象构造函数的影响，你应该直接调用该对象的析构函数，示例如下

      ```C++
      // 假设以下函数用来分配及释放shared memory中的内存
      void *mallocSharedMemory(size_t size);
      void freeSharedMemory(void *memory);

      void *sharedMemory = mallocSharedMemory(sizeof(Widget));
      Widget *pw = new (sharedMemory) Widget();
      ...
      delete pw; // 未定义！因为sharedMemory并非来自operator new

      pw->~Widget(); // 可！析构pw所指的对象，但并未释放sharedMemory
      freeSharedMemory(sharedMemory); // 释放sharedMemory，但未调用任何析构函数
      ```

- 截止到目前为止所考虑的都是单一对象，但是面对数组时，我们需要考虑的会更多一些，例如，当有`string *ps = new string[10];`时，会发生什么事情呢
  - 上述使用的`new`仍然是那个`new` operator，但由于诞生的是数组，所以它的行为会略有不同，它会调用一个名为`operator new[]`的函数来分配内存，和它的兄弟`operator new`一样，`operator new[]`函数也可以被重载
  - “数组版”和“单一对象版”的`new` operator还有一个不同是调用的构造函数的数量不同，前者会针对数组中的每一个对象调用构造函数
  - 同样的道理，当`delete` operator被用于数组时，它会针对数组中每个元素调用析构函数，然后再调用`operator delete[]`来释放内存

## Exceptions

### 9. Use destructors to prevent resource leaks

- 当你在和heap objects打交道时，必须格外的小心资源泄露问题，这个问题不仅会在你忘记手动释放资源时发生，也极有可能在异常出现时发生，考虑下面代码示例

  ```C++
  while(dataSource) { // 假设有个数据源，每次循环都会从中读取一些数据
    Processor *p = readDataSource(dataSource);
    p->process();
    delete p; // 如果在process()中抛出异常，而程序未终止，p将不会被释放
  }
  ```

  - 现在需要考虑，如果`process()`中抛出异常，当前这个函数却未捕捉并处理这个异常，那么异常就会传播到外层调用端进行处理，而当前函数内位于`p->process()`之后的所有语句都会被跳过，这将会导致难以察觉的资源泄露问题
  - 要避免这个问题的办法之一便是使用`try`、`catch`语句块，但是这会将代码逻辑路线搞得乱七八糟，你可能会被迫重复撰写被正常路线和异常路线共享的清理代码，这会造成程序的维护困扰
  - 但是其实有更好的办法，那就是将“一定要执行的清理代码”移到函数内某个局部对象的析构函数中即可，因为局部对象总是会在函数结束时被析构，无论函数如何结束，而C++刚好为我们提供了可以处理这种情况，且行为类似指针的对象，被我们称为smart pointers

- 只要坚持将资源封装在对象内，通常便可以在exceptions出现时避免泄露资源，但还有一些事情需要探讨
  - 如果exceptions是在你正取得资源的过程中抛出的，例如在一个“正在抓取资源”的class constructor内抛出了异常，会发生什么事情呢？
  - 如果exceptions是在此类资源的自动析构过程中抛出的，又会发生什么事情呢？

### 10. Prevent resource leaks in constructors

- 假设现在需要设计一个class用来放置通信簿的数据，其代码如下，此处的代码虽然看似都是inline函数，但是请忽略这个问题，我们只关注它们的行为

  ```C++
  class BookEntry {
  public:
    BookEntry(const std::string &name,
              const std::string &address = "",
              const std::string &imageFileName = "",
              const std::string &audioClipFileName = "")
    : theName(name), theAddress(address), theImage(0), theAudioClip(0) {
      if (imageFileName != "") {
        theImage = new Image(imageFileName);
      }
      if (audioClipFileName != "") {
        theAudioClip = new AudioClip(audioClipFileName);
      }
    }

    ~BookEntry()
    {
      delete theImage;
      delete theAudioClip;
    }
    ...
  private:
    std::string theName; // 个人姓名
    std::string theAddress; // 个人地址
    Image *theImage; // 个人照片，假设用Image类来标识
    AudioClip *theAudioClip; // 一段个人语音，假设用AudioClip类来标识
  };
  ```

  - 这段代码中的构造函数和析构函数看起来都合情合理，确保了正常情况下不会发生资源泄露，而且C++保证删除null指针是安全的，所以析构函数也不必检查指针是否为null，但是在不正常的情况下，即构造函数出现异常的情况下呢？
  - 假设在初始化theAudioClip的时候有异常被抛出，异常可能来源于`operator new`无法分配足够内存，也可能来源于AudioClip的构造函数，不论如何，该由谁来删除theImage已经指向的对象呢，你可能期望析构函数来帮你完成这个工作，但是答案是BookEntry的析构函数绝不会被调用
    - C++只会析构已构造完成的对象，对象只有在其构造函数执行完毕后才算完全构造妥当，这么做的原因是，如果析构函数被作用于尚未完全构造好的对象上时，它如何知道该做哪些部分的事情呢？
    - 如果希望析构函数知道，那就必须在对象内的那些数据身上附带某种指示，指示构造函数进行到了何种程度，那么析构函数就可以检查指示并理解应该如何应对，但是这些额外开销的代价是很高的
  - 另一种你可能会想到的办法是深度参与异常处理，即在对象构造处捕捉异常，部分代码可能如下
    - 很遗憾的是资源还是会泄露，因为除非new操作成功，否者上述的赋值操作并不会施加于pb身上，所以如果BookEntry的构造函数抛出异常，pb将成为null指针，所以此时在`catch`语句块中删除pb除了让你感觉安心之外别无他用

    ```C++
    BookEntry *pb = 0;
    try {
      pb = new BookEntry(name, address, imageFileName, audioClipFileName);
    } catch (...) {
      delete pb; // 捕捉异常，删除pb
      throw; // 异常传递给调用者
    }
    delete pb; // 正常情况下删除pb
    ```

- 为了解决这个问题，方法之一是精心设计构造函数，使他们在异常情况下自我清理，通常只需要将所有可能的异常捕捉起来，执行清理工作，然后重新抛出异常即可，BookEntry的构造函数可以修改如下

  ```C++
  BookEntry(const std::string &name,
            const std::string &address = "",
            const std::string &imageFileName = "",
            const std::string &audioClipFileName = "")
  : theName(name), theAddress(address), theImage(0), theAudioClip(0)
  {
    try {
      if (imageFileName != "") {
        theImage = new Image(imageFileName);
      }
      if (audioClipFileName != "") {
        theAudioClip = new AudioClip(audioClipFileName);
      }
    } catch (...) { // 捕捉所有异常，然后执行必要的清理工作
      delete theImage; // 你可能注意到catch内的清理语句和析构函数内相同
      delete theAudioClip; // 所以较好的做法是把共享代码抽出放进一个辅助函数内
      throw;
    }
  }
  ```

  - 无需担心class内的non-pointer成员变量，因为此处使用了成员初始化列表，所以它们会在构造函数被调用之前就初始化完毕，所以对象被销毁时，其所含的这些成员变量会像“构造完全的对象”一样被自动销毁
    - 但是如果这些对象的构造函数调用其他函数，而那些函数可能抛出异常，那么这些构造函数就必须负责捕捉异常，并在继续传播它们之前执行必要的清理工作
  - 不过该方法还有一种情况无法应对，那便是针对常量指针的初始化，这样的指针必须通过成员初始化列表加以初始化，但是这时我们将无法借助于构造函数内的`catch`语句块来解决问题，因为成员初始化列表只接受表达式（expression）
    - 既然无法将异常处理语句放入成员初始化列表，那么一个可能的地点就是放在private member functions内，让成员变量在其中获得初值

    ```C++
    BookEntry(const std::string &name,
              const std::string &address = "",
              const std::string &imageFileName = "",
              const std::string &audioClipFileName = "")
    : theName(name), theAddress(address),
      theImage(initImage(imageFileName)),
      theAudioClip(initAudioClip(audioClipFileName)) {}

    // theImage首先被初始化，所以即使初始化失败也无需担心
    Image *initImage(const std::string &imageFileName)
    {
      if (imageFileName != "") return new Image(imageFileName);
      else return 0;
    }
    // theAudioClip第二个被初始化，所以如果它初始化期间有异常抛出，则必须执行清理操作
    AudioClip *initAudioClip(const std::string &audioClipFileName)
    {
      try {
        if (audioClipFileName != "") return new AudioClip(audioClipFileName);
        else return 0;
      } catch (...) {
        delete theImage;
        throw;
      }
    }
    ```

    - 虽然该方法完美解决了我们的问题，但是本该由构造函数完成的动作现在却散布于数个函数中，毫无疑问造成了维护上的困扰

- 而一个更好的解答是，接受条款9的忠告，将这两个成员变量所指对象视为资源，交给局部变量来管理，这样的设计下，如果theAudioClip初始化期间有任何异常抛出，已经是个完整构造好的对象，所以它会自动销毁，此外，由于它们如今都是对象，当其“宿主”BookEntry被销毁时，它们亦将自动销毁，所以析构函数中的清理工作也就不再需要了

  ```C++
  class BookEntry {
  public:
    BookEntry(const std::string &name,
              const std::string &address = "",
              const std::string &imageFileName = "",
              const std::string &audioClipFileName = "")
    : theName(name), theAddress(address)
      theImage(imageFileName != "" ? new Image(imageFileName) : 0),
      theAudioClip(audioClipFileName != "" ? new AudioClip(audioClipFileName) : 0) {}
    ...
  private:
    ...
    const auto_ptr<Image> theImage;
    const auto_ptr<AudioClip> theAudioClip;
  };
  ```

### 11. Prevent exceptions from leaving destructors

- 两种情况下析构函数会被调用
  - 第一种是当对象正常情况下被销毁，也就是离开了它的生存空间（scope）或是明确地被删除
  - 第二种是当对象被异常处理机制（也就是异常传播过程中的stack-unwinding，即栈展开机制）销毁

- 因此有两个好的理由支持我们“全力阻止异常传出析构函数之外”
  - 当析构函数被调用时，可能有一个异常正在作用之中，于是你必须在保守的假设下（假设当时有个异常正在作用中）撰写析构函数，因为如果控制权基于异常的因素离开析构函数，而此时正有另一个异常处于作用状态，C++会调用terminate函数来终止程序，甚至不等局部对象被销毁
  - 另一个理由是，如果异常从析构函数内抛出，那个析构函数便是执行不全的，这意味着它没有完成其应该完成的每一件事情，例如应该释放的资源由于异常抛出的原因没有被释放

### 12. Understand how throwing an exception differs from passing a parameter or calling a virtual function

- C++特别声明，一个对象被抛出作为exception时，总是会发生复制（copy），即使是此exception以引用方式被捕捉，如果exception objects以按值方式捕捉，它们甚至会被复制两次

  ```C++
  void passAndThrowWidget(Widget w)
  {
    static Widget localWidget; // static变量的生存空间是整个程序
    cin >> localWidget; // 以引用方式传递参数
    throw localWidget; // 总是会对localWidget进行复制，然后将副本抛出
  }
  ```

  - 而当一个对象被当作一个exception进行复制时，复制行为是由对象的copy constructor执行的，这个copy constructor相应于该对象的“静态类型”而非“动态类型”，是的，这和其他所有C++复制对象的情况一致，复制动作永远是以对象的静态类型为本
    - 基于这一事实，下面两段语句块所做的事情就有了不同，前者是重新抛出当前的exception，后者抛出的是当前exception的副本
      - 而这一差异带来的结果是，前者总是重新抛出当前的exception，不论其类型为何，更明确地说如果最初抛出的exception的类型是SpecialWidget（Widget的派生类），则前者就会传播一个SpecialWidget exception，甚至虽然`w`的静态类型是Widget，这是因为此exception被重新抛出时，并没有发生复制行为
      - 后者语句块则重新抛出一个新的exception，其类型总是widget，因为那是`w`的静态类型，所以一般而言你总是应该使用第一种抛出方式

      ```C++
      catch (Widget &w) { throw; } // 重新抛出当前的exception
      catch (Widget &w) { throw w; } // 抛出的是当前exception的副本
      ```

- “被抛出成为exceptions”的对象，其被允许的类型转换动作，比“被传递到函数”的对象要少
  - 我们知道C++中允许隐式类型转换，但是一般而言，如此的转换并不发生于“exceptions与`catch`语句相匹配”的过程中，例如抛出的int exception绝不会被“用来捕捉double exception”的`catch`语句捕捉到
  - “exceptions与`catch`语句相匹配”的过程中，仅有两种转换可以发生
    - 一种是“继承架构中的类型转换”，即一个针对base class exceptions而编写的异常捕捉语句，可以处理derived class exceptions
    - 另一种是从“有型指针”到“无型指针”的转换，所以一个`const void*`指针可以捕获任何指针类型的exception

- `catch`语句以其“出现于源代码的顺序”被编译器检验比对，这些比对中第一个匹配成功的会被执行，也就是`catch`语句总是依照其出现的顺序进行匹配尝试，因此如果你将针对base class而设计的`catch`语句放在针对derived class而设计的`catch`语句之前，那么后者永远不会得到执行
  - 而当我们以某对象调用一个虚函数时，被选中执行的是那个“与对象类型最佳吻合”的函数，不论它是不是源代码所列的第一个，进一步解释就是当你调用一个虚函数时，被调用的函数是“调用者的动态类型”中的函数，采用所谓的“best fit”策略，而异常处理遵循所谓的“first fit”策略

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
