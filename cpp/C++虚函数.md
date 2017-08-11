# C++虚函数表

C++通过继承和虚函数表来实现多态。多态简单来说就是用基类的指针或者引用绑定到子类的实例，通过运行时决议，使得通过该指针或者引用而调用的函数根据绑定的实例而表现出不同的行为。

## 虚函数表

C++运行时决议主要是通过虚函数表来实现。每一个含有虚函数的类（无论是继承来的还是用户自己定义的）都会有一张对应的虚函数表，表中记录了所有虚函数的地址，类的实例内部含有一个虚函数表指针vptr，指向虚函数表。

举例来说
```C++
struct Base {                                                                                                                                       
  int base;                                                                                                                                         
  virtual void print_t() {                                                                                                                          
    std::cout << "Base" << std::endl;                                                                                                               
  }                                                                                                                                                 
};                                                                                                                                                                                                                                                                                                   
struct Derived {                                                                                                                                    
  int derived;                                                                                                                                      
  virtual void print_t() {                                                                                                                          
    std::cout << "Derived" << std::endl;                                                                                                            
  }                                                                                                                                                 
  virtual void print_d() {                                                                                                                          
    std::cout << "another" << std::endl;                                                                                                            
  }                                                                                                                                                 
};

...
// 假设在一台64位的机器上
Derived d;
int64_t *vptr = (int64_t*)&d;
void (*pfun)() = (void (*)())*((int64_t*)*vptr+1);
pfun();             // 输出：another

...
Base b;
Derived d;
Base *pb = &d;
pb -> print_t();    // 输出：Derived
```

如果我们用GDB打印出变量d的内容，可以得到

```
$1 = (Derived) {
  _vptr.Derived = 0x400d28 <vtable for Derived+16>, 
  derived = 0
}
```

可以看到d有一个vtable指针，如果打印出vtable的内容，可以看到：

```
vtable for 'Derived' @ 0x400d28 (subobject @ 0x7fffffffe1f0):
[0]: 0x400b10 <Derived::print_t()>
[1]: 0x400b3c <Derived::print_d()>

vtable for 'Base' @ 0x400dc0 (subobject @ 0x7fffffffe1e0):
[0]: 0x400b3c <Base::print_t()>
```

由于vptr放在了对象内存的开始位置处，所以可以按照上述程序所示直接拿到虚函数的`print_d`的地址而直接调用。

上述的虚函数表的打印信息也可以看到如果子覆盖了基类的虚函数，那么父类虚函数对应的slot会被替换成子类的虚函数。

现在再来说说运行时决的问题，像上述代码所展示的，一个Base指针`pb`绑定到了一个Derived类实例上，在运行时编译器并不知道pb的所指向的内存的内容，也就无法在编译期做出判断，但是在编译期，编译期知道这个虚函数在虚函数表中的索引，于是上述代码在编译被转换为：

```C++
// 伪C++
pb -> _vptr[0]();
```

为了保证上述转换的正确性，上述`Derived::print_t`和`Base::print_t`的索引值必须一样。实际上，如果有一个指向虚函数成员函数的指针，我们打印出它的值，那么得到的就是其索引值，而如果不是虚函数，得到的是实际函数的地址。

## 多重继承

一个子类继承了多少个父类，它就会有多少张虚函数表。

举例来说：

```C++
struct A {                                                                                                                                          
   int a;                                                                                                                                            
   virtual void print_a() {                                                                                                                          
     std::cout << "A" << std::endl;                                                                                                                  
   }                                                                                                                                                 
};                                                                                                                                                  
                                                                                                                                                    
struct B {                                                                                                                                          
  int b;                                                                                                                                            
  virtual void print_b() {                                                                                                                          
    std::cout << "B" << std::endl;                                                                                                                  
  }                                                                                                                                                 
};                                                                                                                                                  
                                                                                                                                                    
struct AB: public A, public B {                                                                                                                     
  int ab;                                                                                                                                           
  virtual void print_ab() {                                                                                                                         
    std::cout << "AB" << std::endl;                                                                                                                 
  }                                                                                                                                                 
};

...
AB ab;

```

打印ab变量的vtbl，可以看到：

```
vtable for 'AB' @ 0x400dd8 (subobject @ 0x7fffffffe1b0):
[0]: 0x400ab8 <A::print_a()>
[1]: 0x400b10 <AB::print_ab()>

vtable for 'B' @ 0x400df8 (subobject @ 0x7fffffffe1c0):
[0]: 0x400ae4 <B::print_b()>
```

可以看到每一个基类都有一张虚表，并且在第一张表中，存放了子类自己定义的虚函数地址。

如果现在子类覆盖了基类的虚函数呢？即

```C++
struct AB: public A, public B {                                                                                                                     
  int ab;                                                                                                                                           
  virtual void print_ab() {                                                                                                                         
    std::cout << "AB" << std::endl;                                                                                                                 
  }

  virtual void print_a() {
      std::cout << "AB A" << std::endl;
  }                                                                                                                   

  virtual void print_b() {
      std::cout <<"AB B" << std::endl;
  }    
};

```
vtable for 'AB' @ 0x400e40 (subobject @ 0x7fffffffe1b0):
[0]: 0x400b3c <AB::print_a()>
[1]: 0x400b10 <AB::print_ab()>
[2]: 0x400b68 <AB::print_b()>

vtable for 'B' @ 0x400e68 (subobject @ 0x7fffffffe1c0):
[0]: 0x400b93 <non-virtual thunk to AB::print_b()>
```

可以看到覆盖的基类的函数地址填到了第一张表中，第二张表中的`print_b`函数会在运行时被指向第一张表的`AB::print_b`（通过thunk技术）
