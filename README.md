# 第四章 Function 语义学（The Semantics of Function）

先举个列子介绍下这章在讲什么：
```
Point3d Point3d::normalize() const {
  register float mag = magnitude();
  Point3d normal;
  normal._x = _x/mag;
  normal._y = _y/mag;
  normal._z = _z/mag;
  return normal;
}

float Point3d::magnitude() const {
  return sqrt(_x * _x + _y * _y + _z * _z);
}
```
调用方式为：
```
Point3d obj;
Point3d *ptr = &obj;

obj.normalize();
ptr->normalize();
```
问调用的时候会发生啥？
c++类中的三种成员函数： static、nonstatic 和 virtual，每种类型被调用的方式都不同。

本章就是在讲各种function是怎么被调用的，编译器会做些什么。本章分下面几个小节：
1. member的各种调用方式
2. virtual functions
3. 指向member function的指针
4. Inline函数

## 4.1 member的各种调用方式

  历史上，现有nonstatic member function, 20世纪80年代有了virtual function，1987年又有了static member function.

### Nonstatic Member Functions（非静态成员函数）

C++ 的设计准则之一：nonstatic member function 至少和一般的 nonmember function 有相同的效率。举个栗子
```
float magnitude3d(const Point3d *_this) { ... }
float Point3d::magnitude() const { ... }
```
其实编译器就会把第二行的函数变为第一行。
编译器会做的：
1.改写函数的 signature（函数原型）以安插一个额外的参数到 member function 中，该额外参数就是 this 指针。
```
// non-const nonstatic member augmentation
Point3d Point3d::magnitude( Point3d *const this )
```
2.将每一个“对 nonstatic data member 的存取操作”改为经由 this 指针来存取
```
{
  return sqrt(this->_x * this->_x + this->_y * this->_y + this->_z * this->_z );
}
```
3.将 member function 重新写成一个外部函数，并对函数名进行“mangling”处理，使其名称独一无二
```
extern magnitude__7Point3dFv(
	register Point3d *const this );
```

### Virtual Member Functions（虚拟成员函数）

如果 normalize() 是一个 virtual member function
```
ptr->normalize();
```
会被转化为：
```
(*ptr->vptr[1])(ptr);
```
如果 magnitude() 也是一个 virtual function，那么其在 normalize() 中的调用将被转换为:
```
register float mag = (*this->vptr[2])(this);
```
由于 Point3d::magnitude() 是在 Point3d::normalize() 中被调用的，而后者已经由虚拟机制确定了实体，所以这里明确调用 Point3d 实体会更有效率。有一句话是“明确的调用操作会压制虚拟机制”。
```
register float msg = Point3d::magnitude();
```

### static member function
如果 normalize() 是一个 static member function
```
obj.normalize();
ptr->nomalize();
```
会被转换为：
```
normalize__7Point3dSFv();
```
Static member function 主要特性就是它没有 this 指针，其次，它还有以下几个次要特性（都是源于主要特性）：
* 它不能直接存取 class 中的 nonstatic member。
* 它不能被声明为 const、volatile 或 virtual。
* 它不需要经由 class object 才被调用。

指向静态方法的指针的类型是不带类的，比如是
```
unsigned int (*) ();
```
而不是
```
unsigned int (Point3d::*) ();
```


## 4.2 Virtual Member Function

有这样的调用
```
pt->z();
```
问：拥有什么样的信息，才能让我们在执行期正确调用z()?
答：
* pt所指的对象类型
* z()函数实体的位置

为了实现这两点，在对象上增加两个member.
1. 用字符串或者数字表示对象类型。
2. 用一个表存放函数实体的地址，并分配每个对象一个指针指向表。

在单一继承的情况下，virtual table布局如下图

图1

图中表明：派生类和基类中的同名虚函数，其索引一定得是一样的。

虽然我们不能提前知道 ptr 所指对象的真正类型，也不知道哪一个 z() 函数实体会被调用，但可以知道的是每一个 z() 函数地址都放在 slot 4 中。因此，调用会转化为：
```
(*ptr->vptr[4])(ptr)；
```

### 多重继承下的virtual functions
有这样的继承关系：
```
class Base1 {
 public:
  Base1();
  virtual ~Base1();
  virtual void   speakClearly();
  virtual Base1 *clone() const;

 protected:
  float data_Base1;
};

class Base2 {
 public:
  Base2();
  virtual ~Base2();
  virtual void   mumble();
  virtual Base2 *clone() const;

 protected:
  float data_Base2;
};

class Derived : public Base1, public Base2 {
 public:
  Derived();
  virtual ~Derived();
  virtual Derived *clone() const;

 protected:
  float data_Derived;
};
```

有如下代码
```
Base2 *pbase2 = new Derived;
```
新的 Derived 对象的地址必须调整，以指向 Base2 subobject
```
Derived *temp = new Derived;
Base2 *pbase2 = temp ? tmp + sizeof(Base1) : 0;
```
只有这样，下面的非多态的调用才能正常运行
```
pbase2->data_Base2;
```

此时程序员要删除 pbase2 所指对象
```
delete pbase2
```
指针必须再调整一次，以指向 Derived 对象的起始处。
```
Derived::~Derived(pbase2+sizeof(Base1))
```

### 引出一个问题：怎样才能确定offset?
换句话说怎样存取offset.

方法1: 扩充virtual table
```
(*pbase2->vptr[1])(pbase2);
```
被改为：
```
(*pbase2->vptr[1].faddr)(pbase2 + pbase2->vptr[1].offset)
```
faddr 为 virtual function 的地址，offset 为 this 指针的调整值。
这个做法的缺点就是对每个 virtual function 的调用操作都有影响，即使不需要 offset 的情况也是如此。

图(手绘)

方法二：trunk
trunk是一小段机器码，这段机器码可以让this指针调整offet，并且执行虚函数。例如：
```
pbase2_dtor_thunk:
  this += sizeof(Base1)
  Derived::~Derived(this)
```
图(手绘)

优点：虚函数表本身没变化，也避免了不需要的offset.

本例多重继承的布局如下图:
图2


对本例而言，有两个 virtual table：
一个主要实体，与 Base1（最左端 base class）共享。
一个次要实体，与 Base2（第二个 base class）有关。

还有两种情况需要调整this指针，其一：指向derived class的指针，调用base2 继承来的virtual funciton。
```
Derived *pder = new Derived;
// pder 必须向前调整 sizeof(Base1) 个 bytes
pder->mumble();
```
其二：问题发生于一个语言扩充性质之下：允许一个 virtual function 的返回值类型有所变化，比如本例中的 clone() 函数
```
Base2 *pb1 = new Derived;
// 调用 Derived* Derived::clone()
// 返回值必须调整，以指向 Base2 subobject
Base2 *pb2 = pb1->clone();
```
第 1 行调用时，pb1 会被调整以指向 Derived 对象的起始地址，从而 clone() 的 Derived 版会被调用，它会传回一个指向 Derived 对象的指针，在这个指针值被指定给 pb2 之前，必须先经过调整，以指向 Base2 subobject。


## 4.4 指向 Member Function 的指针（Pointer-to-Member Functions）
一个指向 member function 的指针，其声明语法如下：
```
double      // return type
(Point::*   // class the function is member
  pmf)      // name of pointer to member
();         // argument list
```
然后这样用：
```
double (Point::*coord)() = &Point::x;
coord = &Point::y;
// 可以这样调用(origin是对象):
(origin.*coord)();
// 也可以这样调用(origin是指针):
(origin->*coord)();
```
会被编译器转化为：
```
(coord)(&origin));
// 或：
(coord)(ptr);
```
使用“member function 指针”，如果不用于 virtual function、多重继承、virtual base class 等情况的话，并不会比使用一个“nonmember function 指针”的成本更高。

### 支持“指向 Virtual Members Functions”之指针
```
float (Point::*pmf)() = &Point::z; // virtual function
Point *ptr = new Point3d;
```
对一个 virtual member function 取地址，所获得的是一个索引值。此处的&Point::z的结果是1。
下面这两种方法都可以用：
```
ptr->z();
(ptr->*pmf)();
```
后者会被编译器处理为：
```
(*ptr->vptr[(int)pmf])(ptr)
```

现在有个问题：不管是不是virtual函数，其函数都可以声明为：
```
float (Point::*pmf)();
```
那如果Point::x()它不是virtual函数，仍使用下面的方法调用
```
float (Point::*pmf)() = &Point::x;
(ptr->*pmf)();
```
此时编译器转化就会有问题，因为&Point::x不是索引值，而是真实的函数地址。因此，编译器定义pmf必须满足：
1.含有两种数值。
2.其数值可以被区别代表内存地址还是索引值

在 cfront 2.0 非正式版中，这两个值被内含在一个普通的指针内，并使用以下技巧识别该值是内存地址还是 virtual table 索引：
```
(((int)pmf)) & ~127) ? 
  (*pmf)(ptr) : (*ptr->vptr[(int)pmf](ptr));
```
这种实现技巧必须假设继承体系中最多只有 128 个 virtual functions.

### 在多重继承之下，指向 Member Functions 的指针
```
struct __mptr {
  int delta; // this指针的offset值
  int index;  // 带有 virtual table 索引.当 index 不指向 virtual table 时，会被设为 -1
  union {
    ptrtofunc faddr;  // nonvirtual member function 地址
    int       v_offset; // 相对于virtual base class的offset值
  };
};
```
这是为了支持多重继承，设计的结构体。
此时操作``` (ptr->*pmf)(); ```变为：
```
(pmf.index < 0) ?
  (*pmf.addr)(ptr) : (*ptr->vptr[pmf.index](ptr));
```
这种方法会让每个调用操作都得付出上述成本。Microsoft 把这项检查拿掉，导入一个 vcall thunk，在此策略下，faddr 要不就是真正的 member function 地址（如果函数是 nonvirtual），要不就是 vcall thunk 的地址（如果函数是 virtual）。

## 4.5 Inline function
关键词 inline 只是一项请求，如果这项请求被接受，编译器就必须认为它可以用一个表达式（expression）合理地将这个函数扩展开。  
编译器同意展开的条件是：
    * 执行成本 < 一般函数调用以及返回的成本
    

### 形参
在 inline 扩展期间，每一个形参都会被对应的实参取代。
```
inline int min(int i, int j) {
  return i < j ? i : j;
}
```

有三种调用方式：
```
inline int bar () {
  int minval;
  int val1 = 1024;
  int val2 = 2048;
  
  /*(1)*/minval = min(val1, val2);
  /*(2)*/minval = min(1024, 2048);
	/*(3)*/minval = min(foo(), bar()+1);
  
  return minval;
}
```
其中inline会扩展为：
```
//(1) 参数直接代换
minval = val1 < val2 ? val1 : val2;
//(2) 实参是一个常量表达式，则在替换之前先完成求值操作
minval = 1024;
//(3) 有副作用，所以导入临时对象
int t1;
int t2;
minval = 
  (t1 = foo()), (t2 = bar() + 1), 
	t1 < t2 ? t1 : t2;
```

### 局部变量
```
inline int min(int i, int j) {
  int minval = i < j ? i : j;
  return minval;
}
```
如果有以下调用：
```
{
  int local_var;
  int minval;
  // ...
  minval = min(val1, val2);
}
```
会转化为：
```
{
  int local_val;
  int minval;
  // 将 inline 函数的局部变量处以“mangling”操作
  int __min_lv_minval;
  minval = 
    (__min_lv_minval = val1 < val2 ? val1 : val2), 
  	__min_lv_minval;
}
```
inline 函数中的局部变量，再加上有副作用的参数，可能会导致大量临时性对象的产生
```
minval = min(val1, val2) + min(foo(), foo() + 1);
```
转化为：
```
// 为局部对象产生临时对象
int __min_lv_minval_00;
int __min_lv_minval_01;
// 为放置副作用值而产生临时变量
int t1;
int t2;

minval = 
  ((__min_lv_minval__00 = 
    val1 < val2 ? val1 : val2),
   	__min_lv_minval__00) +
  ((__min_lv_minval__01 = 
    (t1 = foo()), (t2 = foo() + 1), t1 < t2 ? t1 : t2), 
   	__min_lv_minval__01)
```
如上所述，如果参数带有副作用，或者在一个表达式中多次调用，或是在 inline 函数中有多个局部变量，都会产生临时对象。对于既要安全又要效率的程序，inline 函数提供了一个强而有力的工具，然而，与 non-inline 函数比起来，它们需要更加小心的处理！
