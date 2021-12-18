# Const correctness

**Синтаксис константных ссылок**

* `const type& a_ref = a` - создаёт константную ссылку на переменную `a_ref` типа `type`. `a` может
  не быть `const`, но менять его в любом случае нельзя. Убрать константность у ссылки тоже
  невозможно.

**Невозможность менять константные объекты и поля**

* Константный объект нельзя менять. Точка. Две точки.
* Константные поля у объекта можно инициализировать только через member initialization list, иначе
  получим ошибку компиляции. Менять их, конечно, тоже нельзя. Пример:

```c++
struct X {
    X (int i) : b(i) {}; // ok
    X (int i) { // bad
          b = i;
    }
    const int b;
};
```

**Const-qualifier у методов**

* Методы могут быть константными, тогда они гарантируют, что не будут изменять состояние объекта. В
  таком случае их можно вызывать на константном объекте. 
* Если `const-qualified` метод класса пытается поменять объект - ошибка компиляции (не UB).
* Пример:

```c++
struct X {
    void foo() const {}; // member-function promises NEVER to change *this
    
    void bar() {}; // member-function might change *this
};

X a;
const X& x = a; // const reference to X class
x.foo(); // OK
x.bar(); // won't compile, bar() is not const

```
