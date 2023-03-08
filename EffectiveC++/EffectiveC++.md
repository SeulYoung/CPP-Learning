# Effective C++

## Accustoming Yourself to C++

### 1. View C++ as a federation of languages

- C
- Object-Oriented C++
- Template C++
- STL

### 2. Prefer consts, enums, and inlines to #defines

- `#define`是预处理指令，而不是语言特性
- 对于常量，最好使用`const`对象或者`enum`，而不是`#define`
- 对于形似函数的宏，最好使用`inline`函数，而不是`#define`

### 3. Use const whenever possible

- `const`可被施加于任何作用域内的对象、函数参数、函数返回类型、成员函数本体
- 编译器强制执行bitwise constness，但不强制执行logical constness，可以考虑使用`mutable`关键字实现概念上的常量性
- 当`const`和`non-const`成员函数有着实质等价的实现时，令`non-const`版本调用`const`版本可避免代码重复

### 4. Make sure that objects are initialized before they're used

- 对象的成员变量初始化动作发生在进入构造函数本体之前，构造函数内的动作更准确的应该叫做赋值动作
- class的构造函数最好使用成员初始化列表，成员变量总是以声明顺序初始化
- 为内置型对象进行手工初始化，因为C++不保证初始化他们
- 函数内的static对象被称为local static对象，其他static对象被称为non-local static对象
- C++对于“定义于不同编译单元内的non-local static对象”的初始化顺序是未定义的
  - 因此常用手法是利用Singleton模式将non-local static对象替换为local static对象
  - 这个手法的基础在于：C++保证，函数内的local static对象会在“该函数被调用期间”“首次遇上该对象之定义式”时被初始化

## Constructors, Destructors, and Assignment Operators

### 5. Know what functions C++ silently writes and calls

- 编译器会为class生成default constructor、copy constructor、copy assignment operator、destructor，所有这些函数都是public且inline的
- default构造函数和析构函数会调用base class和non-static成员变量的构造函数和析构函数
- 编译器生成的析构函数是non-virtual的，除非这个class的base class自身声明有virtual析构函数

### 6. Explicitly disallow the use of compiler-generated functions you do not want

- C++11中，可以使用`=delete`关键字显式禁用某个函数
- C++98中，可以将某个函数声明为private且不实现

### 7. Declare destructors virtual in polymorphic base classes

- 带多态性质的base class应该声明virtual析构函数，以确保其derived class的析构函数被调用
- 如果一个class带有任何virtual函数，那么它应该有virtual析构函数

### 8. Prevent exceptions from leaving destructors

- 析构函数不应该抛出异常，如果一个被析构函数调用的函数可能抛出异常，析构函数应捕捉任何异常，然后吞掉异常或者结束程序

### 9. Never call virtual functions during construction or destruction

- derived class对象内的base class成分会在derived class自身的成分被构造之前先行构造妥当
  - 换句通俗的话说，在base class构造期间，virtual函数不是virtual的
  - 更根本的原因是，在derived class对象的base class构造期间，对象的类型是base class，而不是derived class，不止virtual函数会被编译器解析至base class，其运行期类型信息也会把对象视为base class类型
- 解决办法之一是，既然你无法使用virtual函数从base class向下调用，那就在构造期间令derived class将必要的构造信息向上传递至base class构造函数

### 10. Have assignment operators return a reference to *this

- 赋值操作符应该返回一个指向自身的引用，以便于链式赋值

### 11. Handle assignment to self in operator=

- 确保当对象自我赋值时`operator=`有良好行为，其中技术包括几种
  - 比较“来源对象”和“目标对象”的地址，如果相同则直接返回
  - 精心周到的语句顺序，例如先“记住”原有对象，然后“复制”新值，最后“销毁”原有对象
  - 利用copy-and-swap技术
- 确定任何函数如果操作一个以上的对象，而其中多个对象是同一个对象时，其行为仍然正确

### 12. Copy all parts of an object

- 任何时候为derived class对象撰写copying函数，都必须谨慎的将base class部分一并复制，你应该让derived class的copying函数调用相应的base class函数
  - 保证复制所有的local成员变量
  - 保证调用所有base class内适当的copying函数
- 尽管copy构造函数和copy assignment操作符往往有着近似的实现本地，但你绝不应该令两者互相调用
  - 如果你想要消除两者之间重复的代码，建立一个新的成员函数给两者调用，通常是private函数且被命名为init

## Resource Management

### 13. Use objects to manage resources

- 为防止资源泄露，请使用RAII（Resource Acquisition Is Initialization）对象，它们在构造函数中获取资源并在析构函数中释放资源
- 书中介绍可以考虑使用STL库提供的`auto_ptr`等智能指针来管理资源
  - 早期的C++中，`auto_ptr`是唯一的智能指针，但是它有着一些缺陷，例如不支持array、严格所有权的复制等
  - 注意其实C++11已经废弃了`auto_ptr`，建议使用`unique_ptr`或者`shared_ptr`，后者也增加了对array的支持

### 14. Think carefully about copying behavior in resource-managing classes

- 通常的RAII class copying的行为包括以下几种
  - 禁止复制
  - 对底层资源使用“引用计数法”
  - 复制底层资源
  - 转移底层资源的所有权
- 复制RAII对象必须一并复制它所管理的资源，所以资源的copying行为决定RAII对象的copying行为

### 15. Use the same form in corresponding uses of new and delete

- APIs往往要求访问原始资源，因此每一个RAII class都应该提供一个取得其所管理资源的办法
- 对原始资源的访问可能经由显示转换或隐式转换，一般而言显示转换更为安全，但隐式转换更为方便

### 16. Use the same form in corresponding uses of new and delete

- 当你使用`new`关键字时，有两件事发生
  - 一是分配内存（通过名为operator new的函数）
  - 二是针对此内存会有一个或多个构造函数被调用
- 当你使用`delete`关键字时，也有两件事发生
  - 一是有一个或多个析构函数被调用
  - 二是然后内存被释放（通过名为operator delete的函数）
- 如果你在`new`表达式使用了`[]`，那么你也应该在`delete`表达式使用`[]`，反之亦然，不要混用

### 17. Store newed objects in smart pointers in standalone statements

- 以独立语句将newed对象存储于智能指针中
  - 如果不这样做，一旦一个复杂的语句次序被重排，中间某一步的异常被抛出，有可能导致难以察觉的资源泄露

## Designs and Declarations

### 18. Make interfaces easy to use correctly and hard to use incorrectly

- 好的接口很容易被正确使用，不容易被错误使用
  - 促进正确使用的办法包括接口的一致性，以及于内置类型的行为兼容
  - 阻止错误使用的办法包括建立新类型、限制类型上的操作、束缚对象值以及消除用户的资源管理责任
  - tr1::shared_ptr支持定制型删除器，这也可防范“Cross-DLL problem”，可被用来自动解除互斥锁等

### 19. Treat class design as type design

- 在你想设计一个优秀的class之前，你必须首先思考和回答以下问题
  - 新type的对象应该如何被创建和销毁？
    - 这将影响对象的构造函数、析构函数、内存分配函数、内存释放函数
  - 对象的初始化和赋值该有什么样的差别？
  - 新type对象如果被passed by value，意味着什么？
  - 什么是新type的合法值？
    - 对于class的成员变量来说，通常只有有限数据集是有效的
    - 你必须精心设计成员函数的约束条件检查工作，尤其是构造函数和赋值操作符等
  - 新的type需要配合某个继承图系（inheritance graph）吗？
    - 如果需要，你就会受到这些classes的设计约束，你的设计也会影响继承新type的classes
  - 新的type需要什么样的转换？
    - 如果你允许隐式转换，就必须设计相应的类型转换函数（`operator T()`）
    - 如果你只允许显示转换，就必须设计专门负责执行转换的函数
  - 什么样的操作符和函数对新type而言时合理的？
  - 什么样的标准函数应该被驳回？
  - 谁该取用新type的成员？
  - 什么是新type的未声明接口（undeclared interface）？
  - 新type有多么一般化？
    - 或许你并非定义一个新type，而是定义一整个types家族
    - 若果真如此，则你应该考虑定义一个新的class template
  - 你真的需要一个新的type吗？
    - 是否单纯的定义多个non-member函数或template函数更能达到你的目标？
- Class的设计就是type的设计，在你定义新的type之前，请确定你已经考虑过了上述所有问题

### 20. Prefer pass-by-reference-to-const to pass-by-value

- pass-by-value的代价很高，因为它需要复制对象，并递归的复制对象的所有成员
- pass-by-reference不仅效率更高，还可以避免slicing（对象切割）问题
  - 当一个derived class对象以by value方式传递并被当作base class对象时，base class的copy构造函数会被调用，这将导致derived class的特化性质全被切割掉
- pass-by-reference本质是通过指针实现的，因此往往并不适用于内置类型、STL迭代器和函数对象
  - STL迭代器是一种泛型指针，行为与指针类似
  - 函数对象是一种仿函数，行为与函数类似，一般没有构造函数和析构函数

### 21. Don’t try to return a reference when you must return an object

- 函数的返回值，绝对不要执行以下操作
  - 返回pointer或reference指向local stack对象
  - 返回reference指向heap-allocated对象
  - 返回pointer或reference指向local static对象，而有可能需要多个这样的对象

### 22. Declare data members private

- 切记将成员变量声明为private
  - 赋予用户访问数据一致性
  - 可细微划分访问控制
  - 允诺约束条件获得保证
  - 为class作者提供充分的实现弹性

### 23. Prefer non-member non-friend functions to member functions

- 越多的东西被封装，我们改变这些东西的能力就越大，改变时就能影响越少的用户
- 将不同分类的便利函数放在多个头文件内但隶属于同一个命名空间，可以降低编译依赖型，并且可以让用户自由选择需要的函数
- 宁可用non-member non-friend函数替换member函数，这样可以增加封装性、包裹弹性和机能扩充性

### 24. Declare non-member functions when type conversions should apply to all parameters

- 如果你需要为某个函数的所有参数进行类型转换，那么这个函数必须是个non-member函数

### 25. Consider support for a non-throwing swap

- 通常针对对象的做法是将`std::swap`进行模板全特化（total template specialization）
  - 为了与STL容器保持一致，我们令对象实现一个名为swap的public成员函数，然后令特化版本的`std::swap`调用该成员函数
  - 通常我们不被允许改变std命名空间的任何东西，但可以为标准templates制造特化版本
- 对于class templates而言，我们需要声明一个non-member swap模板函数，但是不能将其声明为`std::swap`的特化版本
  - 因为C++只允许对class templates偏特化（partial specialization），而不允许对function templates偏特化
  - 虽然我们声明的swap模板函数不在std空间内，但C++依据argument-dependent lookup规则，仍能调用我们所定义的专属版本
  - 需要注意的是，不要为调用添加额外修饰符，如`std::swap(a, b)`，这将强迫编译器调用std内的swap函数（包括其任何模板特化）
- 现在，让我们对整个形势进行总结
  - 首先如果`std::swap`的缺省实现能够满足你的需求，你不需要做任何额外的工作
  - 但是如果其效率不足（那几乎总是意味着你的class或template使用了某种pimpl（pointer to implementation）手法）
    - 提供一个public swap成员函数，实现高效的swap操作，但是不要让函数抛出异常
    - 在你的class或template所在命名空间提供一个non-member swap，令其调用上述public swap成员函数
    - 如果你正在编写一个class而非template，为你的class特化`std::swap`，令其调用public swap成员函数
    - 最后，如果你调用swap函数，请确定包含一个using声明，令`std::swap`在你的函数内部可见，然后不加任何修饰符地调用swap函数

## Implementations

### 26. Postpone variable definitions as long as possible

- 尽可能延后定义的真正意义是
  - 不仅应该延后变量的定义，直到非要使用变量的前一刻为止
  - 甚至应该尝试延后定义直到能够赋予其“具明显意义之初值”实参为止，这样能够避免构造和析构非必要对象
- 然而对于循环，需要单独考虑两种情况
  - 对于将变量定义于循环外，成本为：一次构造、一次析构、n次赋值
  - 对于将变量定义于循环内，成本为：n次构造、n次析构
  - 因此如果对象的赋值成本低于一组构造和析构的成本，那么应该考虑将变量定义于循环外
  - 否则，应该将变量定义于循环内，这种做法也能够保证变量的作用域不会超出循环体

### 27. Minimize casting

- C++提供四种新式转型
  - `const_cast`通常用来实现常量性移除
  - `dynamic_cast`通常用来执行安全向下转型，它是唯一无法由旧式语法执行的动作，但是可能消耗重大运行成本
  - `reinterpret_cast`意图执行低级转型，实际动作可能取决于编辑器，也表示其不可移植
  - `static_cast`用来执行强迫隐式转换
- C++中单一对象在不同类型的情况下，可能拥有一个以上的地址
  - 例如以“`Base*`指向它”时的地址和以“`Derived*`指向它”时的地址，尤其是在使用多重继承时，这种情况尤为常见
  - 这意味着“由于知道对象如何布局”而设计的转型操作，在不同编译器上可能会有不同的结果
- derived class重写base class的虚函数后，希望调用base class中对应虚函数的方法
  - 切勿以`static_cast<Base>(*this).Func()`的方式调用base class的虚函数
    - 此方法并非在当前对象身上调用base class的虚函数，而是在“当前对象之base class成分”的副本上调用的函数
  - 正确的方法是使用`Base::Func()`，明确的告诉编译器，我要调用的是base class的函数
- `dynamic_cast`存在执行时较大的开销，有两个一般性的做法可以避免它
  - 其一，是使用容器并在其中存储直接指向derived class的指针，如果要处理多种类型，你可能需要多个容器
  - 其二，是设计通过base class中的接口处理“所有可能之派生类的行为”，这样就可以通过base class调用所有派生类的函数
- 尽量避免转型，如果转型时必要的，试着将其隐藏于某个函数背后，用户不必将转型放进自己的代码中
- 宁可使用C++的新式转型，而不是旧式转型，因为新式转型的行为更加明确，有着分门别类的职责

### 28. Avoid returning `handles` to object internals

- 尽量避免返回handles（包括references、指针、迭代器）指向对象内部
  - 这样做可以增加封装性，帮助const成员函数实现其目标
  - 也可以避免对象被销毁时，handle指向的内存被释放，发生悬挂指针（dangling handles）问题

### 29. Strive for exception-safe code

- 带有异常安全的函数会做到：不泄露任何资源、不允许数据败坏
- 异常安全函数提供以下三个保证之一
  - 基本保证：如果异常被抛出，程序内的任何事物仍然保持在有效的状态下，然而程序的现实状态不可预料
  - 强烈保证：如果异常被抛出，程序状态不改变，函数成功就是完全成功，失败会回复到函数调用之前的状态
  - 不抛出保证：承诺绝不抛出异常，因为它们总是能够完成承诺的功能，其实更准确的说法是如果其抛出异常，将是严重错误
- 强烈保证往往能够以copy-and-sweep来实现，但其强烈保证并非对所有函数都可实现或具备现实意义
- 函数提供的异常安全保证通常最高只等于其所调用各个函数的异常安全保证中的最弱者

### 30. Understand the ins and outs of inlining


