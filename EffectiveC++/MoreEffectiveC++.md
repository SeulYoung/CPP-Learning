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
        - 你可能好奇如果采用`delete [] ws`来释放内存会发生什么，答案是不可预期，因为删除一个不是以new operator获得的指针，其结果是未定义的
  - 第二个缺点是，其将不适用于许多template-based container classes，因为在这些模板内几乎总是会产生一个以“模板类型参数”作为类型而架构起来的数组
    - 例如`template<class T> class Array { ... }`，Array的构造函数内可能包含`data = new T[size];`这样的数组产生代码，如果T没有default constructor，那么模板类也就无法使用了
    - 大多数情况下，如果谨慎设计template，可以消除对default constructors的需求，不幸的是许多设计者什么都有，独缺谨慎
  - 第三个考虑点和virtual base classes有关，virtual base classes如果缺乏default constructors，与之合作将是一种痛苦
    - 因为virtual base classes的构造函数自变量必须由派生层次最深的class提供，这就要求其所有的derived classes都必须知道且了解变量的意义，并提供构造函数所需的变量值

- 虽然有以上种种原因，但是仍然不建议添加无意义的default constructors，尽管这可能会对classes的使用带来某种限制，但是也带来了一种保证，即当你使用classes时，你可以预期该对象会被完全初始化，实现上亦富有效率

## Operators

### 5. Be wary of user-defined conversion functions

### 6. Distinguish between prefix and postfix forms of increment and decrement operators

### 7. Never overload `&&`, `||`, or `,`

### 8. Understand the different meanings of new and delete

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
