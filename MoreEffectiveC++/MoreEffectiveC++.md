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
      // 函数调用过程中将一个临时对象传递给non-const reference参数是不允许的
      // 但是如下代码你可能注意到一些不同，还记得异常抛出总是会发生复制吗？异常传播可以用by reference的方式捕捉被抛出的对象（必为临时对象）
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

- 首先让我们考虑catch by pointer，理论上将一个exception从抛出端搬移到捕捉端是一个缓慢的过程，而by pointer应该是最有效率的一种做法，因为throw by pointer是唯一在搬移“异常相关信息”时不需复制对象的做法（见条款12）
  - 看起来很美好，但是程序员面临着如何让exception objects在控制权离开那个“抛出指针”的函数之后依然存在，或许你会希望使用heap-based对象，但是这样做的代价是昂贵的，而且你必须时刻面对资源泄露的问题
  - 而且catch by pointer和语言本身建立起来的惯例也有所矛盾，4个标准的exceptions统统都是对象，而不是指针，所以你无论如何必须以by value或by reference的方式捕捉它们

- 其次是catch by value，这种方式可以消除上述catch by pointer所面临的部分问题，但是此情况下，每次exception objects被抛出，都需要复制两次，而且会面临到对象切割（slicing）的问题
  - 复制两次的代价体现在，例如有`catch (Widget w) ...`，一次构造动作是“任何exceptions都会产生临时对象”身上，另一次构造动作是“将临时对象复制到`w`”身上
  - 对象切割是因为derived class exception objects被捕捉并视为base class exception objects，这将失去其派生成分，如此被切割过的对象其实就是base class objects，他们缺少derived class data members，当虚函数在其上被调用时会被解析为base class的虚函数（这和对象以by value方式传递给函数时所发生的事情一样）

- 最后就是catch by reference，这种方式不需要考虑指针对象的删除问题，也不会面临对象切割问题，且只会被复制一次，所以毫无疑问这就是你想要的方式

### 14. Use exception specifications judiciously

- 对exception specifications保有持平的观点至为重要，在将它们加入函数之前，请考虑所带来的程序行为是否真的是你所想要的
  - 它们对于函数“希望抛出什么样的exceptions”提供了卓越的说明，而且在“违反exception specifications以至于需要立刻结束程序的悲惨”情况下，它们也提供了`set_unexpected`允许你指定默认行为
  - 但是它们也有一些缺点，包括编译器只对它们做局部性检测，因此很容易被不经意的违反，此外它们可能会妨碍更上层的exception处理函数处理未预期的exceptions

### 15. Understand the costs of exception handling

- 为了能够在运行时期处理exceptions，程序必须做大量簿记工作，exceptions的处理需要成本，即使你从未使用关键词try、throw或catch，你可能也必须付出至少某些成本
  - 在每一个执行点，它们必须能够确认“如果发生exception，哪些对象需要析构”，它们必须在每一个try语句块的进入点和离开点做记号，针对每个try语句块它们必须记录对应的catch子句及能够处理的exceptions类型，这些簿记工作必须付出代价
  - 运行时期的比对工作（以确保符合exception specifications）不是免费的，exception被抛出时销毁适当对象并找出正确的catch子句也不是免费的。

- 认识到异常背后的成本，但是却也不要过度敏感，为了将异常相关的成本最小化，只要能够不支持异常，编译器便不支持，你也需要将异常相关的使用限制于非用不可的地点，并且在真正异常的情况下才抛出exceptions

## Efficiency

### 16. Remember the 80-20 rule

- 80-20法则所表达的重点在于：软件的整体性能几乎总是由其构成要素（代码）的一小部分决定的
  - 当你希望找到瓶颈所在时，避免使用猜测的方法，无论是用经验猜，还是用直觉猜，因为程序的性能特质倾向高度的非直觉性
  - 可行之道就是完全根据观察或实验来识别出那20%的代码，而辨识之道就是借助某个程序分析器

### 17. Consider using lazy evaluation

- lazy evaluation是一种技术，它可以让你延迟计算某些值，直到它们真正被需要为止，毕竟从效率的观点来看，最好的运算是从未被执行的运算，毕竟这不花费任何时间，lazy evaluation可在多种场合派上用场，此处描述四种用途

- 引用计数（Reference Counting）
  - 例如字符串拷贝`String s2 = s1;`，常见的做法就是调用`new` operator分配heap内存，然后再将s1的数据复制到s2所分配的内存中，其实此时s2尚未真正需要实际内容，因为s2尚未被使用
  - lazy evaluation可以省下许多工作，我们让s2分享s1的值，而不再给予s2一个“s1的内容副本”，但是需要做的就是一些记录工作，让我们知道谁共享了什么东西
  - 数据共享的唯一危机是在其中某个字符串被修改时发生，此时应该只有一个字符串被修改，因此此时我们再不能做任何拖延了，必须将s2的内容做一个副本，这样就可以安全地修改了
  - 这种“数据共享”的观念便是lazy evaluation，在真正需要之前，不必着急为某物做一个副本，取而代之的是使用拖延战术，只要还能够，就使用其他副本，如果足够幸运，你可能永远不需要为其提供一个副本

- 区分读和写
  - 考虑有一个字符串`String s = "hello";`，且有对其操作`cout << s[3];`和`s[3] = 'x';`，第一个动作用来读取字符串的某部分，第二个动作则执行一个写入动作
  - 当我们使用reference-counted字符串时，毫无疑问我们希望能够区分两者，因为读取动作代价十分低廉，但是写入动作却可能需要为其先做出一个副本
  - 那么我们是否能够区分`opeartor[]`是在读或写的环境下被调用呢？答案很残忍，我们无能为力，然而如果运用lazy evaluation和条款30描述的proxy classes，我们可以延缓决定“究竟是读还是写”，直到能够确定其答案为止

- 缓式取出（Lazy Fetching）
  - 假设现在程序中有一个大型对象`class LargeObject {...};`，其中包含许多字段，该对象存储于数据库中，现在考虑从磁盘中恢复一个LargeObject所需的成本，此成本可能极高，尤其是这些数据必须从远程数据库中取出时，而后续使用中可能只有少数字段被访问，所以恢复其他字段的成本就是浪费
  - 此问题的lazy evaluation做法是，在产生LargeObject对象时，只产生该对象的“外壳”，不从磁盘读取任何数据，当某个字段被需要时，程序才从数据库中取回对应的数据，下面是一个代码示例

    ```C++
    class LargeObject {
    public:
      LargeObject(ObjectID id);
      const string& field1() const;
      int field2() const;
      ...

    private:
      ObjectID id_;
      mutable string *field1_; // mutable保证了字段在const member function中也能被修改
      mutable int *field2_;
      ...
    };

    LargeObject::LargeObject(ObjectID id)
      : id_(id), field1_(0), field2_(0), ...
    {}

    const string& LargeObject::field1() const
    {
      if (!field1_) {
        // 从数据库中取出field1_
      }
      return *field1_;
    }
    ```

    - 由于LargeObject内的指针，所以我们不得不面对一个问题，就是在使用之前必须对其进行测试，防止指针是无效的，但幸运的是如此单调乏味的苦工可由smart pointer（条款28）自动完成

- 表达式缓评估（Lazy Expression Evaluation）
  - lazy evaluation的这一例子更多用于数值应用，例如有矩阵`Matrix<int> m3 = m1 * m2;`，不用说，这个乘法运算的成本是很高的，因此不如我们记录下这个乘法运算（例如由两个指针和一个enum构成的数据结构），直到真正需要结果时再进行计算，甚至有可能因为程序逻辑更改执行路线，我们再也不需要计算此结果了
  - 另一个更常见的例子是`cout << m3[4];`，我们只需要大型计算中的部分运算结果，而不是整个结果，虽然此时无法再采用拖延战术，但是也不要过度热心，因为没理由在此刻计算第四行之外的任何值，幸运的话，也许根本不必计算它们
  - 但是由于必须存储数值间的相依关系，而且必须维护一些数据结构以存储数值、相依关系，或是两者的组合，此外还必须将赋值、复制、加法等操作符进行重载，所以要达成lazy evaluation的目的，在数值运算领域有许多工作要做

- 尽管lazy evaluation在许多领域都有用途，但是其并非永远都是个好主意，如果你的计算是必要的，其并不会为你的程序节省任何工作或任何实践，甚至可能使程序变慢，并增加内存用量，所以在使用lazy evaluation之前，你必须先考虑清楚，是否真的需要它
  - 一个常见的策略是，先行使用直接易懂的eager evaluation策略，在分析报告指出“此class乃性能瓶颈所在”之后，以另一个实行lazy evaluation的class替换之

### 18. Amortize the cost of exoected computations

- 本条款提出你可通过over-eager evaluation（超急评估）如caching（缓存）和prefetching（预先取出）等做法分期摊还运算成本
  - 这和条款17并不冲突，当你必须支持某些运算而其结果不总是需要的时候，lazy evaluation可以改善程序效率
  - 但当你必须支持某些运算而其结果总是几乎被需要时，或其结果常常被多次需要时，over-eager evaluation可以改善程序效率
  - 两者都比最直接了当的eager evaluation难以实现，但是两者都能为适当的程序带来巨大的性能提升

- Caching（缓存）的使用示例之一便是，假设有一个相对昂贵的数据库查询动作，而你确信查询出的该数据会被频繁使用，那么便可以用相对廉价的“内存内数据结构查找动作”取代之，该策略就是使用一个局部缓存，这个缓存应该可以降低查询一次数据的平均成本

- Prefetching（预先取出）做法的经验是，如果某处的数据被需要，通常其邻近的数据也会被需要，这便是有名的locality of reference现象，系统设计者依此现象而设计出了磁盘缓存（disk caches）、指令与数据的内存缓存（memory caches）、指令预先取出（instruction prefetches）
  - 你可能觉得这些略显遥远，那么更常见的使用示例便是动态数据的扩容动作，此处我们应该使用over-eager evaluation，理由是，如果我们必须增加数组的大小以容纳新元素，locality of reference建议我们未来或许还需再增加大小
  - 因此为避免第二次扩张所需的内存分配成本，可以把数组的大小调整到比它目前所需大小更大一些，希望未来的扩张落入我们此刻所增加的弹性范围内

### 19. Understand the origin of temporary objects

- C++中真正的所谓临时对象是不可见的，不会在你的源代码中出现，只要你产生一个non-heap对象而没有为它命名，便产生了一个临时对象，此等匿名对象通常发生于两种情况
  - 一是当隐式类型转换被施行以求函数能够调用成功时，注意只有当对象以by value方式传递或以by reference-to-const方式传递时，才会发生隐式类型转换
  - 二是当函数返回对象时，这种代价在观念上难以避免，但是有时候你可以以某种方式撰写返回值为对象的函数，使编译器得以将临时对象优化，最常见也最有用的就是被称为return value optimization（RVO）的策略
  - 切勿将函数中的局部对象和临时对象混为一谈

### 20. Facilitate the return value optimization

- ISO/ANSI标准委员会宣布，命名对象和匿名对象都可以借由return value optimization（RVO）被优化去除

### 21. Overload to avoid implicit type conversions

- 隐式转换虽然方便，但是此类转换所产生的临时对象会带来一些我们并不想要的成本，那么为了消除类型转换的需求，我们的做法就是声明数个函数，每个函数有不同的参数表，利用函数重载来消除类型转换
  - 不过请不要忘记80-20法则，增加一大堆重载函数也不见得是件好事，除非你有理由相信，使用重载函数后，程序的整体效率可获得重大改善

### 22. Consider using `op=` instead of stand-alone `op`

- C++并未在`operator+`、`operator=`和`operator+=`之间设立任何互动关系，因此要确保你所期望的互动关系，必须自己实现，一个好方法就是以复合形式（如`operator+=`）为基础实现其独身形式（如`operator+`）

  ```C++
  class Rational {
  public:
    ...
    Rational& operator+=(const Rational& rhs);
    Rational& operator-=(const Rational& rhs);
  };

  const Rational operator+(const Rational& lhs, const Rational& rhs)
  {
    return Rational(lhs) += rhs;
  }

  // 如果你不介意把独身形式操作符放在全局范围，甚至可以利用template消除撰写必要
  template <typename T>
  const T operator-(const T& lhs, const T& rhs)
  {
    return T(lhs) -= rhs;
  }
  ```

- 另一个就是操作符的效率问题，一般而言，复合操作符比其对应的独身形式更有效率，因为后者通常必须返回一个新对象，因此必须负担一个临时对象的构造和析构成本，而复合操作符则是将结果写入其左端自变量，因此不需要额外的临时对象，因此作为程序库设计者，你应该两者都提供，以便用户可以根据需要自行选择

### 23. Consider alternative libraries

- 有时，不同的程序库即使提供相似的机能，也往往表现出不同的性能取舍策略，所以如果你发现某个程序库的性能不尽如人意，你可以考虑是否有可能改用另一个程序库而移除某些瓶颈，由于不同程序库将效率、扩充性、移植性、类型安全性等不同的设计具现化，有时候你可以找找看是否存在另一个功能相近的程序库，而其在效率上有较高的设计权重，如果有，改用它或许可大幅改善程序性能

### 24. Understand the costs of virtual functions, multiple inheritance, virtual base classes, and RTTI

- C++编译器必须找出一种方法来实现语言中的每一个性质，这种细节因编译器而异，大部分时候你并不需要关心这件事，然而某些语言特性的实现可能会对对象的大小和其member functions的执行速度带来冲击，所以面对这类特性，了解“编译器可能以什么样的方法来实现它们”是件重要的事情

- 这类性质中最重要的就是虚函数，当虚函数被调用时，执行的代码必须对应于“调用者的动态类型”，大部分编译器是通过使用所谓的virtual tables（vtbls）和virtual table pointers（vptrs）来提供这样的行为的
  - vtbl通常是一个由“函数指针”架构而成的数组或链表，程序中每一个class若声明或继承虚函数，都会有一个vtbl，其中的条目就是该class的各个虚函数实现体的指针
    - 因此虚函数的第一个成本就是你必须为每个拥有虚函数的class耗费一个vtbl空间，其大小视虚函数的个数而定，每个class应该只有一个vtbl，因此vtbls占用的空间通常不是很大，除非你在每个class内都拥有大量虚函数
  - vptr则是用来指向vtbl的指针，凡声明虚函数的class，其对象都包含一个隐藏的data member（即vptr），该数据被放在只有编译器知道的位置，不同编译器可能放在不同的地点
    - 这也引出了虚函数的第二个成本，你必须在每个拥有虚函数的对象内付出一个额外指针的代价，如果对象不大，这份额外的开销则可能形成值得注意的成本

- 现在来考虑虚函数的调用，假设有class C2继承class C1，当有虚函数调用`pC1->f1()`时，如果只看这个片段，因为多态的存在，无法知道C1和C2中哪一个f1该被调用，因此编译器必须完成以下动作
  - 根据对象的vptr找出其vtbl，这是一个简单的动作，成本只有一个偏移调整（以便获得vptr）和一个指针间接动作（以便获得vtbl）
  - 找出被调用函数在vtbl内的对应指针，这也很简单，因为编译器为每个虚函数指定了独一无二的表格索引，成本只是一个offset以求进入vtbl数组
  - 最后一步就是调用对应的函数，可以大致想象一下最终产出的代码将是`(*pC1->vptr[i])(pC1);`，含义为调用`pC1->vptr`所指的vtbl中的第i个条目所指函数，pC1被传给该函数作为`this`指针之用
  - 可以看出这几乎和一个非虚函数效率相当，只需数个指令就可完成，但是虚函数真正的运行期成本发生在和inline互动时，由于虚函数意味着直到运行期才知道哪个函数被调用，因此编译器几乎没有能力将虚函数加以inlining
    - 这便是虚函数的第三个成本，你事实上等于放弃了inlining（如果虚函数通过对象被调用，倒是可以inlined，但是这并不是虚函数使用的常态）

- 截至目前的每件事情，既适用于单一继承也适用于多重继承，但是多重继承会让事情变得更加复杂
  - 此时对象内会有多个vptrs（每个base class各对应一个），而且针对base classes而形成的特殊vtbls也会被产生出来
  - 当多重继承面对virtual base classes的需求时，又会多出一个或多个指向“virtual base class成分”的指针，这些指针是为了消除多条继承路径时data members的复制现象

- 最后一个和多态相关的成本便是运行时期类型辨识（runtime type identification, RTTI）成本，RTTI让我们能够在运行期获得objects和classes的相关信息，所以一定有某些地方用来存储这些信息才行
  - 这些信息被存放在类型为`type_info`的对象内，你可以利用`typeid`来取得某个class对应的`type_info`对象
  - C++规范书上说，只有当某种类型拥有至少一个虚函数时，才保证我们能够检验该类型对象的动态类型，这使得RTTI相关信息听起来有点像vtbl，而其设计理念也确实是根据class的vtbl来实现的
    - 例如vtbl数组内，索引为0的条目可能内含一个指针，指向“该vtbl所对应class”的`type_info`对象，因此RTTI的空间成本并不太可能为你招惹麻烦

- 以下表格对继承相关的成本做了一份摘要
  | 性质 | 对象大小增加 | Class数据量增加 | Inlining几率降低 |
  | ---- | ------------ | --------------- | --------------- |

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
