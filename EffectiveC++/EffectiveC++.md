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
  - 更根本的原因是，在derived class对象的base class构造期间，对象的类型是base class，而不是derived class
  - 不止virtual函数会被编译器解析至base class，其运行期类型信息也会把对象视为base class类型
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
- 尽管copy构造函数和copy assignment操作符往往有着近似的实现本体，但你绝不应该令两者互相调用
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

- 让我们从封装开始讨论，首先，越多的东西被封装，越少人可以看到它，我们改变这些东西的能力就越大，改变时就能影响越少的用户
  - 现在考虑对象内的数据，越少代码可以看到数据，越多的数据被封装，我们就能越自由的改变对象数据，通过计算能够访问该数据的函数数量就能作为一种粗糙的量测，因此成员变量应该是private的，否则就有无限量的函数可以访问它们
  - 因此，现在让你在member函数和non-member函数之间做选择，两者提供完全相同的机能，那么导致较大封装性的毫无疑问是non-member non-friend函数，因为其并不增加“能够访问class内private成分”的函数数量
- 在C++中，比较自然的做法是让该类函数成为non-member函数并位于相关联class所在的同一个namespace
  - 另一个优势是，将不同分类的便利函数放在多个头文件内但隶属于同一个命名空间，可以降低编译依存性，并且可以让用户自由选择需要的函数
- 现在，你应该能够理解这种反直觉的行为，即宁可用non-member non-friend函数替换member函数，这样可以增加封装性、包裹弹性和机能扩充性

### 24. Declare non-member functions when type conversions should apply to all parameters

- 假设现有一个class Number，该class允许“int-to-Number”隐式转换，你需要为其实现一个`operator*`操作
  - 此时你的直觉告诉你应该保持面向对象精神，将其实现为成员函数，写法为`const Number operator* (const Number& rhs) const`
    - 很快你就会发现，当你尝试混合式算术时只有一半行得通，即`Number * 2`行得通，但`2 * Number`却会出错，如果你以函数形式重写两式为`Number.operator*(2)`和`2.operator*(Number)`，问题一目了然
    - 结论是，只有当参数被列于参数列表内，这个参数才是隐式类型转换的合格参与者，而“被调用之成员函数所隶属的那个对象”，即`this`对象这个隐喻参数，绝不是隐式转换的合格参与者，这也解释了为什么`2 * Number`为什么会出错，此时你并能指望编译器自动将数字隐式转换为Number，然后调用`operator*`成员函数
  - 最终，可行之道拨云见日，让`operator*`成为一个non-member函数，这允许编译器在每一个实参身上执行隐式转换，写法为`const Number operator* (const Number& lhs, const Number& rhs)`
- 最后，请记住，如果你需要为某个函数的所有参数（包括被this指针所指的那个隐喻参数）进行类型转换，那么这个函数必须是个non-member函数

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

- 不要过度热衷inline函数，因为其会造成代码膨胀，从而可能导致额外的换页行为，降低指令高速缓存的击中率
- inline函数只是对编译器的建议，并非强制命令，大部分编译器拒绝将太过复杂的函数inlining，而所有对virtual函数的调用（除非是最平淡无奇的）也都会使inlining落空，因为virtual意味着直到运行期才能确定调用哪个函数，而inline意味着执行前先将调用动作替换为被调用函数的本体
  - 隐喻声明inline函数的方式是将其定于于class的定义式内，明确的inline函数的方式是在函数定义式前加上inline关键字
  - 有时候编译器虽有意愿inlining某个函数，还是可能为函数生成一个本体，例如程序要获得某个inline函数的地址，编译器通常必须为此函数生成一个函数本体
- 值得注意的是，inline函数和template函数通常都被定义于头文件中，但是template函数并非一定是inline的
  - inline函数通常一定被置于头文件内，因为大多数编译器为了将“函数调用”替换为“被调用函数本体”，必须知道函数定义式，因此inlining大多是编译期行为
  - template通常也被置于头文件内，因为它一旦被使用，编译器为了将其具现化，同样必须知道函数定义式
  - 但是template的具现化与inlining无关，如果你认为某个template函数应该被inlining，请明确的将其声明为inline
- 构造函数和析构函数通常是inline的糟糕候选人，因为编译器可能以精致复杂的代码来实现对象创造和销毁时的各种保证，而这些代码可能就放在你的构造函数和析构函数中

### 31. Minimize compilation dependencies between files

- 编译器依存性最小化的本质正是在于“声明的依存性”替换“定义的依存性”，尽量让头文件自我满足，如果做不到则让它与其他文件内的声明式（而非定义式）相依赖
- 如果使用object references或object pointers可以完成任务，就不要使用objects
  - 你可以只靠一个类型声明式就定义出指向该类型的references和pointers，但如果定义了某类型的objects，就必须用到该类型的定义式
- 如果能够，尽量以class声明式替换class定义式
  - 注意，当声明一个函数而它用到某个class时，并不需要该class的定义，纵使函数以by value方式传递类型的参数（或返回值）
  - 或许你会惊讶为何不需要知道定义细节，但事实是，一旦任何人调用这些函数，调用之前class的定义式就必须先曝光才行
- 为声明式和定义式提供不同的头文件
  - 这两个文件必须总是保持一致性，而程序库的用户总是应该include那个声明式的头文件而非前置声明若干函数
  - 只含声明式的头文件命名方式参考C++标准程序库头文件`<iosfwd>`，其包含iostream各组件的声明式
- 通常利用Handle classes和Interface classes解除接口和实现之间的耦合关系，从而降低文件间的编译依存性
  - 代价是运行期丧失若干速度，又为每个对象超额付出若干内存
  - 对于Handle classes，成员函数必须通过implementation pointer取得对象数据，且必须初始化并指向动态分配而来的object
  - 对于Interface classes，由于每个函数都是virtual，所以每次函数调用必须付出间接跳跃的开销，且派生对象必须包含一个vptr（virtual pointer table）

## Inheritance and Object-Oriented Design

### 32. Make sure public inheritance models "is-a"

- public继承意味is-a关系，适用于base class身上的每一件事情一定也适用于derived class，因为每一个derived class对象也都是一个base class对象

### 33. Avoid hiding inherited names

- derived class内的名称会遮掩base class内的名称，在public继承下从来没有人希望如此
- 为了让被遮掩的名称再见天日，可使用using声明式或forwarding function
  - using声明式的语法是`using base::func;`，其作用是让base class内所有名为的func的函数在derived class作用域内可见并且public
  - forwarding function的语法是`type func(parameter list) { return base::func(parameter list); }`，现在只有某个对应参数版本的func函数在derived class中才可见

### 34. Differentiate between inheritance of interface and inheritance of implementation

- 接口继承和实现继承不同，在public继承下，derived class总是继承base class的接口
- pure virtual函数只具体指定接口继承，impure virtual函数具体指定接口继承及其缺省实现继承
- non-virtual函数具体指定接口继承及其强制性实现继承

### 35. Consider alternatives to virtual functions

- 不妨考虑virtual函数的替代方案
  - 使用non-virtual interface（NVI）手法，这是Template Method设计模式的一种特殊形式，它以public non-virtual成员函数包裹较低访问性（private或protected）的virtual函数
  - 将virtual函数替换为函数指针成员变量，这是Strategy设计模式的一种分解表现形式
  - 将继承体系内的virtual函数替换为另一个继承体系内的virtual函数，这是Strategy设计模式的传统实现手法
- 将机能从成员函数移到class外部函数，带来的缺点是，非成员函数将无法访问class的non-public成员

### 36. Never redefine an inherited non-virtual function

- non-virtual函数是静态绑定（statically bound），而virtual函数是动态绑定（dynamically bound）

### 37. Never redefine a function's inherited default parameter value

- virtual函数是动态绑定（dynamically bound），而缺省参数值却是静态绑定（statically bound）
  - 静态绑定又名前期绑定，动态绑定又名后期绑定
  - 对象的静态类型（static type）是其在程序中被声明时所采用的类型
    - 如`Shape* ps = new Circle;`，`ps`的静态类型是`Shape*`
  - 而对象的动态类型（dynamic type）则是指目前所指向对象的实际类型，也就是说动态类型可以表现出一个对象实际将会有什么行为，动态类型可在程序执行过程中改变（通常是经由赋值操作）
    - 如`Shape* ps = new Circle;`，`ps`的动态类型是`Circle*`
- 至于C++为何会以这种方式运作，答案在于运行期效率，如果缺省参数值是动态绑定的，编译器就必须有某种方法在运行期为virtual函数决定适当的参数缺省值
- 合适的做法是考虑virtual函数的替代设计，其中之一便是NVI手法，令base class内的一个public non-virtual函数调用private virtual函数，后者可被derived class重新定义

### 38. Model "has-a" or "is-implemented-in-terms-of" through composition

- 复合（composition）是类型之间的一种关系，当某种类型的对象内含它种类型的对象，便是这种关系
- 在应用域（application domain），即程序中的对象相当于你所塑造的世界中的某些事物中，复合意味着has-a
- 在实现域（implementation domain），即对象纯粹是实现细节上的人工制品，例如缓冲区、互斥器等，复合意味着is-implemented-in-terms-of

### 39. Use private inheritance judiciously

- private继承规则
  - private继承时，编译器不会自动将一个derived class对象转换为base class对象
  - private继承时，base class中的所有成员，在derived class中都会变成private属性
- private继承意味着is-implemented-in-terms-of（根据某物实现出），其用意是为了采用base class内已经备妥的某些特性，而不是对象之间存在任何观念上的关系，private继承纯粹只是一种实现技术
- private继承虽然意味着is-implemented-in-terms-of，但它的优先级要低于复合（composition），你需要明智而审慎的使用private继承，但是当derived class需要访问protected base class的成员，或需要重新定义继承而来的virtual函数时，这么设计是合理的
- 和复合不同的是，private继承可以实现empty base最优化，这对致力于对象尺寸最小化的程序库开发者而言，可能很重要

### 40. Use multiple inheritance judiciously

- 多重继承（multiple inheritance）是指一个derived class可以继承自多个base class，但是相应的，base class中相同的名称可能会产生歧义性，要解决这个问题你必须明白的指出要调用哪一个base class内的函数
- 多重继承的base classes如果在继承体系中又有着共同且更高级的base class，则会产生更致命的菱形继承问题
  - 此时你需要面对这样一个问题，如果顶层base class中存在一个成员变量A，那么你是否打算让base class内的成员变量经由每一条路径被复制？如果是，那么最底层的derived class内将会有两份A成员变量，但简单的逻辑告诉我们，derived class中不应该有两份A成员变量
- C++在这场多重继承的辩论中并没有倾斜立场，两个方案其都支持
  - 缺省情况下的做法是执行复制，也就是会产生两份成员变量
  - 如果那不是你想要的，则必须令那个带有此数据的base class成为virtual base class，你必须令所有直接继承自它的classes采用virtual继承
- 从正确行为的观点看，public继承应该总是virtual继承的，但是不要盲目使用virtual继承，因为你需要为virtual继承付出代价
  - 使用virtual继承的class所产生的对象往往比使用non-virtual继承的体积大
  - 访问virtual base class的成员变量时，也比访问non-virtual成员时速度慢
  - 支配virtual base class初始化的规则远为复杂且不直观，virtual base的初始化责任是由继承体系中最底层的class负责
    - 因此class若派生自virtual bases而需要初始化，必须认知其virtual bases，不论那些bases距离多远
    - 当一个新的derived class加入继承体系时，它必须承担其virtual bases（不论直接或间接）的初始化责任
- 因此对virtual base class的使用忠告很简单
  - 第一，非必要不使用virtual base class，必须确定，你的确是在明智而审慎的情况下使用它
  - 第二，如果必须使用，尽可能避免在其中放置数据，如果virtual base class不带任何数据，将是最具实用价值的情况

## Templates and Generic Programming

### 41. Understand implicit interfaces and compile-time polymorphism

- C++中面向对象编程的世界总是以显式接口（explicit interface）和运行期多态（runtime polymorphism）解决问题
  - 对class而言接口是显式且以函数签名为中心的，多态则通过virtual函数发生于运行期
- 但Template和泛型编程的世界，与面向对象有根本上的不同，此世界中则是以隐式接口（implicit interface）和编译期多态（compile-time polymorphism）解决问题
  - 对template参数而言，接口是隐式的，奠基于有效表达式，多态则是通过template具现化和函数重载解析（function overloading resolution）发生于编译器
- “运行期多态”和“编译期多态”类似于“哪一个重载函数该被调用（发生在编译期）”和“哪一个virtual函数该被绑定（发生在运行期）”之间的差异

### 42. Understand the two meanings of typename

- 从C++的角度看，声明template参数时，不论使用关键字`class`还是`typename`，意义完全相同
- template内出现的名称如果相依于某个template参数，称之为从属名称（dependent names），如果从属名称在class内呈嵌套状，则称之为嵌套从属名称（nested dependent name），如果某个名称不依赖于任何template参数，则称之为非从属名称（non-dependent names）
  - 嵌套从属名称有可能导致解析困难，假设现在有`template<typename C> ...`，同时有`C::const_itrator* x;`（注意这并非有效代码），看起来似乎是声明`x`为一个local指针变量
  - 但是，如果`C`有个static成员变量而碰巧被命名为`const_iterator`，或如果`x`碰巧是个global变量名称呢？那样的话上述语句就变成了一个相乘动作（这听起来确实有点疯狂，但C++解析器必须操心所有可能的输入）
  - 而C++有个规则可以解析此起义状态：如果解析器在template中遭遇一个嵌套从属名称，便假设这名称不是个类型，除非你告诉它，因此缺省情况下嵌套从属名称并非类型
- 请使用关键字`typename`标识嵌套从属类型名称，但不得在base class lists（基类列）或member initialization list（成员初值列）内以它作为base class修饰符

### 43. Know how to access names in templatized base classes

- C++往往拒绝在templatized base classes（模板化基类）内寻找继承而来的名称，那是因为base class templates有可能被特化，而特化版本可能不提供和一般性template相同的接口
  - 就某种意义而言，当我们从Object Oriented C++跨进Template C++时，继承就不像以前那般畅行无阻了
- 为了使C++不进入templatized base classes观察的行为失效，有三种办法
  - 第一，在base class函数调用动作之前加上`this->`
  - 第二，使用using声明式，告诉编译器进入base class作用域内查找
  - 第三，使用`Base<type>::func()`明确指出被调用的函数位于base class内，但这往往是最不让人满意的一个解法，因为如果被调用的是virtual函数，这种明确资格修饰（explicit qualification）会关闭virtual绑定行为
- 从名称可视点（visibility point）的角度出发，上述每一个解法做的事情都相同，对编译器承诺“base class template的任何特化版本都将支持其一般泛化版本所提供的接口”，但如果这个承诺最终未被实践，往后的编译最终还是会面临失败

### 44. Factor parameter-independent code out of templates

- Templates生成多个classes和多个函数，所以任何template代码都不该与某个造成膨胀的template参数产生相依关系
- 因非类型模板参数（non-type template parameters）而造成的代码膨胀，往往可消除，做法是以函数参数或class成员变量替换template参数
- 因类型参数（type parameters）而造成的代码碰撞，往往可降低，做法是让带有完全相同二进制表述（binary representations）的具现类型共享实现码
  - 例如大多数平台上，所有指针类型都有相同的二进制表述，因此templates持有指针者（如`list<int*>`和`list<SquareMatrix<long, 3>*`等等）往往应该对每一个成员函数使用唯一一份底层实现
  - 如果你实现某些成员函数而它们操作强类型指针（strongly typed pointers，即T*），你应该令它们调用另一个操作无类型指针（untyped pointers，即void*）的函数，由后者完成实际工作

### 45. Use member function templates to accept "all compatible types"

- C++中，derived class指针可以隐式转换为base class指针，指向non-const对象的指针可以转换为指向const对象的指针等等，但是对于模板代码`SmartPtr<Base> p1 = SmartPtr<Derived>(new Derived)`呢？
  - 显然，如果以带有继承关系的两个class为模板参数分别具现化某个template，产生出来的两个具现体并不带有base-derived关系，所以编译器视两者为完全不同的classes
- 面对这个问题，显然一个template可以被无限量的具现化，如果我们寄希望于编写构造函数来实现转型的需要，那我们需要的构造函数数量就无止尽了，因此我们需要一个构造模板，这样的模板是所谓的member function templates，作用是为class生成函数
  - 假设现有模板类`template<typename T> class SmartPtr { ... }`，则其构造模板的一般形式为`template<typename U> SmartPtr(const SmartPtr<U>& other);`
  - 此代码的意思是，对任何类型T和任何类型U，可以根据`SmartPtr<U>`生成一个`SmartPtr<T>`对象
  - 上述泛化copy构造函数并未声明为explicit，因为原始指针类型之间的转换是隐式转换，所以仿效这种行为也属合理
- 但是还有另一个问题需要解决，那就是我们希望根据`SmartPtr<Derived>`创建`SmartPtr<Base>`，但是却不希望根据`SmartPtr<Base>`创建`SmartPtr<Derived>`，或根据`SmartPtr<double>`创建`SmartPtr<int>`，因此我们必须从某方面对这一member template所创建的成员函数群进行拣选或筛除
  - 假设SmartPtr内有原始指针成员变量`T* heldPtr;`，则可以用`template<typename U> SmartPtr(const SmartPtr<U>& other) : heldPtr(other.getPtr()) { ... }`来实现代码中约束转换行为
  - 使用成员初始列表来初始化`SmartPtr<T>`之内类型为`T*`的成员变量，并以类型为`U*`的指针作为初值，这个行为只有当“存在某个隐式转换可将`U*`指针转换为`T*`指针”时才能通过编译
- member function template（成员函数模板）的效用不限于构造函数，常用的另一方面是支持赋值操作的兼容行为
- member function template是个奇妙的东西，但它们并不改变语言的基本规则，例如在class内声明泛化copy构造函数（一个member template）并不会阻止编译器生成它们自己的copy构造函数（一个non-template）
  - 因此如果你想要控制copy构造的方方面面，必须同时声明泛化copy构造函数和正常的copy构造函数，相同的规则也适用于赋值操作

### 46. Define non-member functions inside templates when type conversions are desired
