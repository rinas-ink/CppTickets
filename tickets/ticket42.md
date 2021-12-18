## Билет 42. Полиморфные классы

* ## Полиморфные классы, виртуальные функции, `override` `final` для методов, отличия от hiding, вызовы виртуальных функций

  Виртуальная функция - функция, объявленная в базовом классе и переопределенная в наследниках. Создается с помощью ключевого слова `virtual`. Наследники могут перезаписать её поведение словом override.

  До использования виртуальных функций столкнулись с такой проблемой: (не смотря на то что bd указывает на derived мы вызываем функцию из base тк колмпилятор решает какую функцию вызвать только на основании информации полученной на момент компиляции - видит, что ссылка на бэйс и вызывает функцию оттуда). Исправляется просто с помощью virtual.

  ```c++
  #include <iostream>
  #include <vector>
  
  struct Base {
      int x = 10;
  
      /*virtual*/ void print() const {
          std::cout << "x = " << x << "\n";
      }
  };
  
  struct Derived : Base {
      void print() const /*override /* C++11 */ {  // override: добавить virtual, проверить, что в родителе virtual есть. На самом деле virtual добавляется автоматически, если был в родителе.
          std::cout << "x = " << x << ", y = " << y << "\n";
      }
  };
  
  int main() {
      Base b;
      Derived d;
      b.print();
      d.print();
  
      Base &db = d;
      db.print();
  
      std::cout << sizeof(Base) << ", " << sizeof(Derived) << "\n";
  }
  ```

  Класс полиморфный, если у него есть хотя бы 1 виртуалная фнукция.

  * #### `override`

    Проверяет что эта функция действительно перезаписывает какую-то функцию из родителя.

    override можно не писать (но можно случайно не перезаписать функцию), если функция совпадает с какой-то сверху - она вирутальная.

  * #### `final`

    Запрещает наследование для классов:

    ```c++
    struct Derived1 final : Base {
        int value = 123;
    };
    struct SubDerived : Derived1 {}; // ban
    ```

    А еще и наследование методов:

    ```c++
    struct Base {
        virtual void foo() = 0;
        virtual void bar() = 0;
    };
    
    struct Derived : Base {
        void foo() final {  // final implies 'virtual'
        }
    
        // 'override' is redundant:
        // void fooo() final {}  // CE, which is fine.
        // virtual void fooo() final {}  // Not CE, which is not fine.
        virtual void fooo() final override {}  // CE, which is fine.
    
        void bar() override {
        }
    };
    
    struct SubDerived : Derived {
        // 'override' is not important, will not compile either way.
        // void foo() override {}
    
        void bar() override {
        }
    };
    ```

    `final override` - избыточно

  * #### отличия от hiding

    Отличия: Виртуальные функции нужны, чтобы вызываться из самого вложенного класса, а hiding про перегрузки - ищем наиболее подходящую функцию.

    [Из лекции про hiding](https://youtu.be/8-7duHce3Bo?list=PL8a-dtqmQc8obAqSKqGkau8qiafPRCxV7&t=1384)

    4 этапа правила поиска нужной функции:

    1. Name resolution (поиск по имени) Output: "overload set". (множество перегрузок)

    2. Overload resolution. Output: a single overload. Выбираем наиболее подходящую перегрузку

    3. Access check. Определяем можно ли трогать.

    4. Call, can be virtual or non-virtual. Вызываемся, смотря на виртуальность.

       Мы смотрим на все перегрузки foo, которые есть в структуре и только на них. На родителя мы не смотрим и не находим функции, которые нам подходят. (Если в структуре вообще нет нужной функции - тогда идем в родителя). 

    Если хотим перегрузки от родителя, то есть два способа:

    1. руками все сделать
    2. сделать unhiding: в теле класса написать `using <BaseClass>::<FunctionName>`.

    Мем: unhiding <<наследуется>> в смысле того, что пока еще раз не сделать hiding, то все перегрузки будут доступны.

    ```c++
    struct Base {
        void foo() {
            std::cout << "no args\n";
        }
        void foo(int) {
            std::cout << "int\n";
        }
    };
    
    struct Derived : Base {};
    
    struct SubDerived : Derived {
        void foo(double) {
            std::cout << "double\n";
        }
    
        // Will hide, no compilation error.
        // void foo(int x) { std::cout << "SubDerived int\n"; Derived::foo(x); }
        // void foo() { std::cout << "SubDerived no args\n"; Derived::foo(); }
    
        // Introduce members of Base/Derived called 'foo' into the definition, "unhide".
        // using Derived::foo;
        // using Base::foo;
    };
    
    struct SubSubDerived : SubDerived {
        // Has the same overloads as Derived, unless we add new overloads:
        void foo(int, int, int) {}
        using Derived::foo;
    };
    
    int main() {
        Base b;
        b.foo(1);    // int
        b.foo(1.2);  // int
        b.foo();     // no args
    
        SubDerived sd;
        sd.foo(1.2);  // double
        sd.foo(1);    // double :(
        sd.foo();     // compilation error?
        // Rule: if the derived class has a method called `foo`, do not look at
        //       base's methods, "hide" them.
    
        Derived &d = sd;
        d.foo(1);    // int
        d.foo(1.2);  // int
    
        SubSubDerived ssd;
        ssd.foo(1.2);  // int, because of 'using Derived::foo' instead of 'using SubDerived::foo'
    }
    ```

  * #### Вызовы виртуальных функции

    > Егор: есть указатель на таблицу виртуальных функций, мы в него пошли - посмотрели. - низкий уровень
    >
    > Высокий - посмотрели куда ссылается ссылка или указатель. Вызвали оттуда метод. 

* ### Один из способов реализации: таблица виртуальных функций, в том числе с наследованием

  От `std::function` до `vtable` :

  Наивная реализация полиморфизма:

  ```c++
  struct Base {
  int x = 10;
  
      std::function<void()> print = [&]() {
           std::cout << "x = " << x << "\n";
       };
       // std::function<void()> pretty_print = ....;
       // std::function<void()> read = ....;
  };
  struct Derived : Base {
      int y = 20;
  
      Derived() {
          print = [&]() { std::cout << "x = " << x << ", y = " << y << "\n"; };
      }
  };
  ```

  Проблема: `std::function` занимает много памяти (где-то 32 байта) и линейно растет  размер структуры с ростом количества функций.

  Заменим `std::function` на указатель на функцию `void (*)(Base *)`  (который может указывать только на глобальную или статическую функцию, на функци.-члены класса он указывать не может - поэтому функцию сделаем статической и будем явно принимать параметр Base*).  Указатель на функцию маленький. Занимает 4 или 8 байт в зависимости от архитектуры.

  В чем разница? Эта лямбда внутри себя хранит не только код, но еще и запоминает this - что избыточно, мы и так знаем, в каком объекте мы лежим.

  Обычно создают псевдоник для указателя на функцию, потому что тип страшный.   `using print_impl_ptr = void(*)(Base*)`

  Создаем в Base и  Derived отдельную статическую функцию print_impl, параметр у которой - явно переданный объект Base *.  Дальше возьмем эту функцию и сохраним её в поле типа указатель на функцию.

  Но print_ptr нужно явно передавать, на каком объекте мы вызваем. Поэтому желаем такую обычную функцию print, которая вызовет print_ptr на this

  ```c++
   struct Base {
   int x = 10;
  
   using print_impl_ptr = void(*)(Base*);
   static void print_impl(Base *b) {
       std::cout << "x = " << b->x << "\n";
   }
  
   print_impl_ptr print_ptr = print_impl;
   // pretty_print_impl_ptr pretty_print_ptr;
   // read_impl_ptr read_ptr;
  
   void print() {
       print_ptr(this);
   }
  };
  
  struct Derived : Base {
      int y = 20;
      static void print_impl(Base *b) {
          Derived *d = static_cast<Derived*>(b);
          std::cout << "x = " << d->x << ", y = " << d->y << "\n";
      }
      Derived() {
          print_ptr = print_impl;
      }
  };
  // вызов все еще x.print();
  ```

  У нас все еще есть линейный рост.

  Оптимизируем это.

  Пусть у нас много функций и мы сохранили много полей. Но все поля либо указывают на реализации из Base, либо из Derived. Не может быть такого, что какая-то часть на Base, другая - на Derived. Заведем 2 таблицы с этими полями. В структурах будем хранить указатель на табличку, которую мы используем.

  `Vtable` - таблицы виртуальных функций
  В каждой структуре храним указатель таблицы виртуальных функций `BaseVtable *vptr = &BASE_VTABLE;`

  Независимо от количества виртуальных функций размеры структур не меняются. Растет размер только vtable.

  ```c++
  struct Base;
  
  struct BaseVtable {  // virtual functions table
      using print_impl_ptr = void(*)(Base*);
      print_impl_ptr print_ptr;
      // pretty_print_impl_ptr pretty_print_ptr;
      // read_impl_ptr read_ptr;
  };
  
  struct Base {
      static const BaseVtable BASE_VTABLE;
  
      BaseVtable *vptr = &BASE_VTABLE;
      int x = 10;
  
      static void print_impl(Base *b) {
          std::cout << "x = " << b->x << "\n";
   }
  
      void print() {
          vptr->print_ptr(this);
      }
  };
  const BaseVtable Base::BASE_VTABLE{Base::print_impl};
  struct Derived : Base {
      static const BaseVtable DERIVED_VTABLE;
      int y = 20;
  
  static void print_impl(Base *b) {
      Derived *d = static_cast<Derived*>(b);
      std::cout << "x = " << d->x << ", y = " << d->y << "\n";
  }
  
  Derived() {
      vptr = &DERIVED_VTABLE;
  }
  };
  const BaseVtable Derived::DERIVED_VTABLE{Derived::print_impl};
  ```

  Что если хотим добавить в derived новые виртуальные функции? В данной реализации: нужно заводить новую табличку, хранить указатели на старую и новую таблички. Размер Derived растет линейно с уровнем вложенности.
  Решение: наследуем виртуальные таблицы друг от друга. Сделаем структуру `struct DerivedVtable : BaseVTable` и туда добавим новые поля.

  ```c++
  struct Base;
  struct BaseVtable {
      using print_impl_ptr = void(*)(Base*);
      print_impl_ptr print_ptr;
  };
    
  struct Base {
      static BaseVtable BASE_VTABLE;
      
      BaseVtable *vtr = &BASE_VTABLE;
      int x = 10;
        
      static void print_impl(Base *b) {
          std::cout << "x = " << b->x << "\n";
      }
        
      void print() {
          vtable->print_ptr(this);
      }
  };
  BaseVtable Base::BASE_VTABLE{Base::print_impl};
    
  struct Derived;
  struct DerivedVtable : BaseVtable {
      using mega_print_impl_ptr = void(*)(Derived*);
      mega_print_impl_ptr mega_print_ptr;
  };
    
  struct Derived : Base {
      static DerivedVtable DERIVED_VTABLE;
    
      int y = 20;
        
      static void print_impl(Base *b) {
          Derived *d = static_cast<Derived*>(b);
          std::cout << "x = " << d->x << ", y = " << d->y << "\n";
      }
        
        static void mega_print_impl(Derived *b) {
            std::cout << "megapring! y = " << b->y << "\n";
        }
        
        Derived() {
            vptr = &DERIVED_VTABLE;
        }
        
        void mega_print() {
            static_cast<DerivedVtable*>(vtable)->mega_print_ptr(this);
        }
    };
    DerivedVtable Derived::DERIVED_VTABLE{Derived::print_impl, Derived::mega_print_impl};
    
    struct SubDerivedVtable : DerivedVtable {
        // no new "virtual" functions
    };
    struct SubDerived : Derived {
        static SubDerivedVtable SUBDERIVED_VTABLE;
    
        int z = 20;
        
        static void mega_print_impl(Derived *b) {
            SubDerived *sd = static_cast<SubDerived*>(b);
            std::cout << "megaprint! y = " << sd->y << ", z = " << sd->z << "\n";
        }
        
        SubDerived() {
            vptr = &SUBDERIVED_VTABLE;
        }
    };
    SubDerivedVtable SubDerived::SUBDERIVED_VTABLE{Derived::print_impl, SubDerived::mega_print_impl};
  ```

  Здесь понятно, почему виртуальные вызовы --- это долго. Нам надо ВСЕГДА переходить по указателю, а это долго. А еще можно мазать мимо кэша.

  Кстати, `Vtable` может хранить данные для типа. Например, его название.

- #### Чисто виртуальные функции и абстрактные классы

  Идея: есть семейство объектов, у которых какие-то общие вещи, но нет общей реализации (типа ширина, высота, но формулы нет).

  `virtual int width() const = 0` - чисто виртуальная функция. У нее нет реализации. И мы должны переопредилить в каком-то из наследников.

  Абстрактный класс - есть хотя бы 1 чисто виртуальная функция. (Нельзя создать экзэмпляр такого класса, тк у него нет каких-то методов)

- #### Возможность вызвать чисто виртуальную функцию

  Чисто виртуальную функцию можно использовать в коде от наследников, если она реализована там. 

  ```c++
  #include <iostream>
  
  struct Widget {
      virtual int width() const = 0;  // pure virtual => Widget is 'abstract'
      virtual int height() const = 0;
  
      int area() const {
          return width() * height();
      }
  };
  
  struct Button : Widget {
      std::string label;
  
      Button(std::string label_) : label(std::move(label_)) {}
  
      int width() const override {
          return 10 + 8 * label.length();
      }
  
      int height() const override {
          return 10 + 12;
      }
  };
  
  struct Image : Widget {
      int w, h;
  
      Image(int w_, int h_) : w(w_), h(h_) {}
  
      int width() const override {
          return w;
      }
  
      int height() const override {
          return h;
      }
  };
  
  void print_area(const Widget &w) {
      std::cout << w.area() << "\n";
  }
  
  int main() {
      Button btn("Click Me!");
      Image img(60, 70);
      print_area(btn);
      print_area(img);
      // Widget w;  // CE
      // new Widget();  // CE
  }
  ```

- ## Виртуальный деструктор: когда, зачем, что будет, если не сделать

  [Часть лекции про это](https://youtu.be/_lUC9fJ2fcM?list=PL8a-dtqmQc8obAqSKqGkau8qiafPRCxV7&t=4752)

  Правило: если есть хотя бы 1 виртуальная функция - скорее всего деструктор тоже должен быть виртуальным.

  В наследниках виртуальный деструктор генерируется автоматически.

  ```c++
  struct Widget {
      ...
      virtual ~Widget() = default;  // IMPORTANT! Rule: add virtual dtor in polymorphic base.
      // virtual ~Widget() {};
      
  };
  
  struct Button : Widget {
      ...
      // virtual ~Button() = default; // automatically generated
  
  };
  
  struct Image : Widget { ... };
  
  int main() {
      {
          Button *b = new Button("Click me!");
          delete b;
      }
      {
          Image *i = new Image(60, 70);
          delete i;
      }
      {
          Widget *w = new Button("Click me!");
          delete w;  // Not UB: ~Button() because ~Widget() is virtual
      }
  }
  ```

  Если не виртуальный деструктор:

  ```c++
  Widget *w = new Button("Click me!");
  delete w; 
  ```

  Вызываем деструктор widget, а надо Button. UB - вызов деструктора не того типа.

- ## Запрет копирования и перемещения полиморфных объектов: синтаксиc

  Создаем базовый класс noncopyable. И наследуемся от него. Теперь класс некопируемый.

  ([Лекция](https://youtu.be/GNDBJ3i_JII?list=PL8a-dtqmQc8obAqSKqGkau8qiafPRCxV7&t=3227 ))

  ```c++
  struct noncopyable {  // boost::noncopyable
      noncopyable() = default;
      noncopyable(const noncopyable &) = delete;
      noncopyable(noncopyable &&) = delete;
      noncopyable &operator=(const noncopyable &) = delete;
      noncopyable &operator=(noncopyable &&) = delete;
  };
  
  struct Foo : private noncopyable {};
  struct Bar : private noncopyable {};
  struct Baz : private noncopyable {};
  
  int main() {
      Foo f;
      // noncopyable &n = f;  // WTF, 'private' prevents that.
  
      // Foo f2 = f;
  }
  ```

- ## Вызовы виртуальных функций в конструкторах и деструкторах: обычные, с явным указанием класса через `::`

  Виртуальные функции в коснструкторах и деструкторах вызываются без виртуальности. Рассмотрим в примере констурктор от derived. В нем мы вызываем коструктор от Base. Но на этот момент Derived еще не создан - вызываем foo из base. Потом уже из Derived. Аналогично с деструктором. Сначала уничтожили Derived. Идем уничтожать base, и функцию из derived вызвать уже нельзя.

  [Пример из лекции](https://youtu.be/GNDBJ3i_JII?list=PL8a-dtqmQc8obAqSKqGkau8qiafPRCxV7&t=2371)

  ```c++
  #include <iostream>
  
  struct Base {
      int value = 123;
  
      virtual void foo() {
          std::cout << "foo(" << value << ")\n";
      };
  
      Base() {
          foo(); //Base
      }
  
      ~Base() {
          foo(); //Base
      }
  };
  
  struct Derived : Base {
      int value2 = 456;
  
      void foo() override {
          std::cout << "foo(" << value << ", " << value2 << ")\n";
      }
  
      Derived() : Base(), value2(100) {
          foo(); //Base, Derived
      }
  
      ~Derived() {
          foo(); //Derived, Base
      }
  };
  
  int main() {
      Derived d;
      std::cout << "== constructed ==\n";
      d.foo();
      static_cast<Base &>(d).foo();
      std::cout << "== foo called ==\n";
  }
  ```

  Чисто виртуальные:

  Ошибка компиляции, тк функция printTo( ) не реализована. Но можно реализовать (расскомитеть строчки Base::printTo())

  ```c++
  #include <iostream>
  
  struct Base {
      int value = 123;
  
      virtual void printTo() = 0;
  
      Base() {
          printTo();
      }
  };
  
  /*void Base::printTo() {  // BUT WHY
      std::cout << "Not so pure, eh?\n";
  }*/
  
  struct Derived : Base {
      void printTo() override {
      }
  };
  
  int main() {
      // Base b;  // class is still abstract.
      Derived d;
  }
  ```



- ## `dynamic_cast` для указателей и ссылок (без обработки исключений), проверка `dynamic_cast` внутри `if` одновременно с определением новой переменной, требование на полиморфность класса

  Один из умных кастов: `static_cast` делал down cast, если попали, иначе UB. А `dynamic_cast` кастует, если можно, иначе кидает `nullptr`.

  При этом, чтобы `dynamic_cast<T>` скомпилися нужно, чтобы `T` был полиморфным. В основном используется для проверки можно ли скастовать:

  ```
  if (const SubDerived1 *md1 =
          dynamic_cast<const SubDerived1 *>(&b)) {  // C++03
      std::cout << "SubDerived1 " << md1->value << "\n";
  }
  // md1 is not visible
  
  if (const Derived2 *d2 = dynamic_cast<const Derived2 *>(&b);
      d2 != nullptr) {  // C++17: if with init statement
      std::cout << "Derived2 " << d2->value << "\n";
  }
  // d2 is not visible
  ```

  `dynamic_cast` для `nullptr` всегда возвращает `nullptr` (стандарт).

  Иногда `dynamic_cast` вместе с RTTI отрубают ибо медленно/раздувайте бинарник. Ключ в gcc: `-fno-rtti`

  ### Ссылки

  Второе использование: вместе со ссылками.

  Если нельзя скастовать, то получим исключение, которое можно поймать.



- ## RTTI, оператор `typeid`, типы `type_info` и `type_index`, использование `boost::core::demangle`

  * ### RTTI - run time type information

    > Информация про типы, которая доступна во время выполнения.
    > Полезен для полиморфных классов.

  * ### оператор `typeid`

    [cppreference](https://en.cppreference.com/w/cpp/language/typeid)

    Один из двух базовых операторов, который использует RTTI. Позволяет узнать, что за объект. Требуется именно ссылка, не умный указатель (он скажет, что там `std::unique_ptr<Base, std::defualt_delete<Base>>`)

    1. typeid от какого-то конкретного типа
       `const std::type_info &info_base = typeid(Base);` Компилятор возвращает константную ссылку на некоторую структуру type_info. В ней лежит какая-то информация про тип

       ```
       const std::type_info &info_base = typeid(Base);
       const std::type_info &info_derived = typeid(Derived);
       const std::type_info &info_int = typeid(int);
       
       // Получить имя типа
       std::cout << boost::core::demangle(info_base.name()) << "\n";
       std::cout << boost::core::demangle(info_derived.name()) << "\n";
       std::cout << boost::core::demangle(info_int.name()) << "\n";
       ```

    2. typeid от неполиморфного выражения

       Если класс не полиморфный, например, `int`, то тип будет выведен очень просто.
       (foo() - возвращает инт и эта функция не вызывается вообще)

       ```
       typeid(2 + 2 + foo()) = typeid(int)
       ```

    3. А если полиморфный, то получится прикольно:

       (переменная полиморфного типа, ссылка на полиморфный класс)

       typeid честно вычисляет значение полностью. Компилятор берет ссылку и смотрит, на какой объект эта ссылка на самом деле указывает.

       ```
       std::cout << "===== 2a (polymorphic) =====\n";
           Base b;
           Derived d;
           std::cout << boost::core::demangle(typeid(b).name()) << "\n";
           std::cout << boost::core::demangle(typeid(d).name()) << "\n";
           std::cout << boost::core::demangle(typeid(bar()).name())
                     << "\n";  // bar() is called
       
           Base &bd = d; // именно ссылка, (умный) указатель не сработает
           std::cout << boost::core::demangle(typeid(bd).name()) << "\n";
       ```

    ### операции с typeid

    У `typeid` есть `operator==`:

    ```
    Base b;
    Derived d;
    
    const std::type_info &info_base = typeid(Base);
    const std::type_info &info_derived = typeid(Derived);
    const std::type_info &info_b = typeid(b);
    const std::type_info &info_d = typeid(d);
    
    std::cout << (info_base == info_b) << "\n";     // 1
    std::cout << (info_base == info_d) << "\n";     // 0
    std::cout << (info_derived == info_b) << "\n";  // 0
    std::cout << (info_derived == info_d) << "\n";  // 1
    ```

    Заметим, что названия выводятся криво: `i` vs `int` (gcc vs MSVC) и так далее. Есть `type_index`, который позволяет класть `typeid` в сеты.

  * ### тип `type_info`

    `std::type_info` - класс, который содержит информацию о типе, включая имя типа, средства сравнения, порядок сортировки. Возвращается оператором `typeid` - [cppreference](https://en.cppreference.com/w/cpp/types/type_info)

    Можно брать константную ссылку, name, сравнивать на равно - не равно. Рефлексия.

  * ### тип `type_index`

    Обертка над type_info. Можно копировать, сравнивать, класть в set и unordered set

    [cppreference](https://en.cppreference.com/w/cpp/types/type_index)

  * ### использование `boost::core::demangle`

    Использовали в typeid.

    `std::cout << boost::core::demangle(info_base.name()) << "\n";`

    Эта функция в зависимости от того, каким компилятором мы компилируем пытается догадаться, что хотела сказать функция name и вывести имя типа.

    [Документация](https://www.boost.org/doc/libs/1_63_0/libs/core/doc/html/core/demangle.html)

    `info_base.name()` выводит разное в зависимости от компилятора

- ##### Не было: эмуляция виртуальных операторов (вроде `operator<<`, практика `14-210121`)

- ##### Не было: виртуальные конструкторы и паттерн "фабричная функция" (в том числе для `make_unique` и копирования)
