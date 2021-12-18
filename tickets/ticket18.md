## Билет 18. `unique_ptr`, управление памятью, move
* ## Ручное управление памятью: `new`, `delete`, `new[]`, `delete[]`, когда что использовать.
Это `Dinamic storage duration`, `new` выделяет память, обычно на куче, и возвращает указатель на свежесозданный объект, который мы обязуемся удалить с помощью оператора `delete`. (иначе произойдёт утечка памяти). Квадратные скобки выделяют массив элементов, которые будут лежать подряд в памяти. В скобках нужно указать количество объектов в массиве. При этом при удалении этого делать не нужно.  
```C++
struct Foo{

};
int main() {
    Foo *f = new Foo;
    Foo *arr = new Foo[1000];
    delete f;
    delete[] arr; // no memory leak
}
```
  * ### Не было: разница между `new int;` и `new int();`
  * ### Утечка памяти: UB ли, какие последствия, как ловить, как читать вывод sanitizer и Valgrind с примерами.  
  Утечка памяти это не уб, программа спокойна работает, просто появляется всё новая и новая выделенная память, что может вызвать проблемы. Например, если мы сервер вк, то утечки памяти лучше не допускать, иначи со временем памят на сервере кончится. Ловить с помощью `valgrind` или санитайзеров.  
  ```C++
    struct Foo{
        int a;
    };
    int main() {
        Foo *f = new Foo; // Memory leak
        return 0;
    }
  ```
  Запускаем валгринд командой `valgrind ./a`. Он напишет сколько памяти утекло.  
  ```
    ==135348== Memcheck, a memory error detector
    ==135348== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
    ==135348== Using Valgrind-3.15.0 and LibVEX; rerun with -h for copyright info
    ==135348== Command: ./a
    ==135348== 
    ==135348== 
    ==135348== HEAP SUMMARY:
    ==135348==     in use at exit: 4 bytes in 1 blocks
    ==135348==   total heap usage: 2 allocs, 1 frees, 72,708 bytes allocated
    ==135348== 
    ==135348== LEAK SUMMARY:
    ==135348==    definitely lost: 4 bytes in 1 blocks
    ==135348==    indirectly lost: 0 bytes in 0 blocks
    ==135348==      possibly lost: 0 bytes in 0 blocks
    ==135348==    still reachable: 0 bytes in 0 blocks
    ==135348==         suppressed: 0 bytes in 0 blocks
    ==135348== Rerun with --leak-check=full to see details of leaked memory
    ==135348== 
    ==135348== For lists of detected and suppressed errors, rerun with: -s
    ==135348== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)

  ```  
  Скомпилировать с санитайзером можно вот так `g++-10 main.cpp -o a -fsanitize=address`. Вот что он выдаст  
  ```
    =================================================================
    ==136349==ERROR: LeakSanitizer: detected memory leaks

    Direct leak of 4 byte(s) in 1 object(s) allocated from:
        #0 0x7fce088235a7 in operator new(unsigned long) ../../../../src/libsanitizer/asan/asan_new_delete.cpp:99
        #1 0x5633a583225e in main (/home/mirong/CLionProjects/hse/plusi/ut/untitled/a+0x125e)
        #2 0x7fce0821e0b2 in __libc_start_main (/lib/x86_64-linux-gnu/libc.so.6+0x270b2)

    SUMMARY: AddressSanitizer: 4 byte(s) leaked in 1 allocation(s).
  ```
  
* ## Отличия `unique_ptr` от обычного указателя, когда что использовать  
`unique_ptr` это `smart_pointer`. Это некая обёртка над обычным указателем, добовляющая какую-то семантику, кто указателем владеет, кто удаляет, какие-то операции запрещает делать, каие-то делает автоматически. `unique_ptr` сам чистит за собой память, его не нужно удалять руками. При этом его нельзя копировать, только мувать. Получать значение можно так же, как и у обычного указателя.  
Если вы не хотите возиться с удалением памяти, и вам не нужно копирование указателей, то используйте `unique_ptr`.  Только если вам нужна конкретно семантика владения, иногда нужно `shared_ptr`, `weak_ptr` и т.д.  
* ## Создание `unique_ptr`: какие есть конструкторы, что делает `make_unique`  
Можно просто объявить, тогда это `nullptr`  
```C++
std::unique_ptr<Foo> f;
assert(f == nullptr);
```
Можно собственно записать указатель, с помощью `make_unique`. Под капотом это обычный `new`, только возвращающий сразу `unique_ptr`. `make_unique` передаёт аргументы в конструктор.
```C++
std::unique_ptr<Foo> f = std::make_unique<Foo>();
std::unique_ptr<Foo> f = std::unique_ptr<Foo>(new Foo); // так то же норм, завернули указатель в unique_ptr
```
  * ### Проблема с `make_unique` и приватными конструкторами
  Во первых, факт. Можно френдить функции из других неймспейсов. Хотим создовать через объект класса только через `make_unique`, и имеем только приватный конструктор. Тогда потупому звывать `make_unique<Foo>()` не получится, так как не имеем доступ к конструктору.  
  * ### Решения: неработающее с друзьями, работающее с фабричной статической функцией
Поэтому может возникнуть идея сделать `make_unique<Foo>()` френдом класса. Но она плохая, так как сработает только в том случае, если именно `make_unique` вызовет в себе контруктор. Если же он под копотом делигирует эту операцию в какую-то другую функцию, то ничего не сработает, так как у неё не будет доступа к приватному конструктору. Решение такое - сделать отдельную функцию `make`, в которой мы вызовем `new Foo`, и завернём полученный указатель в `unique_ptr`. (вызвать здесь `make_unique` всё так же не получится).  
```C++
#include <memory>

// https://abseil.io/tips/134

struct Foo {
    static std::unique_ptr<Foo> make() {
        // return std::make_unique<Foo>();  // bad
        return std::unique_ptr<Foo>(new Foo());  // good
    }

private:
    Foo() {}

    // Technically possible, but won't help becase std::make_unique may construct indirectly
    // friend std::unique_ptr<Foo> std::make_unique<Foo>();
};

int main() {
    // auto p1 = std::make_unique<Foo>();  // hence, this is bad
    auto p2 = Foo::make();  // this is good
}  
  
```
* ## Невозможность копирования
`unique_ptr` нельзя копировать, только мувать.  
* ## Синтаксис перемещения: в параметры функции, из функции, в/из других переменных (включая поля: `13-211208/02-move-objects/03-move-to-field`), в/из контейнеров  
Если мы возвращаем локальную переменную из функции, компилятор видит это и мувает автоматически, т.к. локальная переменная в любом случае умрёт после return.  
  
  Принимаем в конструкторе аргумент по значению, а затем муваем его в поле. Таким образом мы разделяем процесс: аргумент разрешаем инициализировать как угодно (самым быстрым образом), а потом бесплатно муваем его в поле и удаляем пустой объект. Итого получили одну потенциально долгую инициализацию и две бесплатных штуки (если, конечно, мув и удаление для такого объекта работают быстро)
  Дальше вставлен код с лекции. Нужно сравнить передачу по константной ссылке и по значению. init, copy, destruct- дорогие операции. move, copy, destruct of empty- дешёвые операции. Единственный сценарий, когда мы не хотим передавать по значению, если объект очень большой, и мувать + удялять его очень дорого. Но такого лучше не допускать.  
  В итоге получается, что передавать переменную выгоднее по констентной ссылке. Временный объект по значению. В третьем случае (где у нас функция возвращает строчку по значениб) так же выгоднее передавать по значению.  
  
```C++
  #include <string>
  #include <utility>

  struct PersonCpp03 {
      std::string name;
      PersonCpp03(const std::string &name_) : name(name_) {}  // 1 copy
  };

  struct PersonCpp11 {
      std::string name;
      PersonCpp11(std::string name_) : name(std::move(name_)) {}  // 1 initialization name_ + 1 move + 1 destruct of empty
  };

  std::string create_name() {
      std::string s = "hello world";
      return s;  // no std::move needed
  }

  int main() {
      {
          std::string x = "Egor";
          [[maybe_unused]] PersonCpp03 p1(x);  // x is copied into p1.name: 1 copy
          [[maybe_unused]] PersonCpp03 p2("Egor");  // temporary is copied: 1 init, 1 copy, 1 destruct
          [[maybe_unused]] PersonCpp03 p3(create_name());  // temporary is copied: 1 init inside create_name(), 1 copy, 1 destruct
      }
      {
          std::string x = "Egor";
          [[maybe_unused]] PersonCpp11 p1(x);  // 1 copy + 1 move + 1 destruct of empty
          [[maybe_unused]] PersonCpp11 p2("Egor");  // 1 init, 1-2 move, 1-2 destruct of empty
          [[maybe_unused]] PersonCpp11 p3(create_name());  // 1 init inside create_name(), 1-3 move, 1-3 destruct of empty
      }
  }
  ```
  * ### Когда (не) надо писать `std::move`
    Не хочется мувать большие объекты лишний раз, если их достаточно просто скопировать (смотри предылущий пример с лекции). Или наоборот, когда проще скопировать объект, чем переставлять указатели (например короткие строки).
* ## `move` как оптимизация для копируемых объектов, автоматическая поддержка `move` у пользовательских структур
  После третьих плюсов, любые пользовательские структуры можно мувать. До этого их приходилось только копировать, при `swap` например. Если копируемый объект нам больше не нужен, то компилятор сам поменяет копирование на `move`.
  * ### moved-from состояние у объектов
  Если мувать из `unique_prt`, то останется гарантированно `nullptr`. Для других оъектов это не гарантируется, например для строк или векторов. Строчка остаётся корректной после мува, то есть можно её вывести, узнать размер. Но о её содержимом у нас никакхи гарантий нет. Это сделано из-за того, что коротки строчки проще скопировать, чем переставлять указатели.  
```C++
#include <cassert>
#include <memory>
#include <string>
#include <vector>
#include <iostream>

int main() {
    {
        std::unique_ptr<int> ptr(new int(200));
        std::cout << ptr.get() << "\n";
        std::move(ptr);  // Does nothing.
        std::cout << ptr.get() << "\n";
        auto ptr2 = std::move(ptr);
        // Guaranteed: ptr.get() == nullptr
        std::cout << ptr2.get() << " " << ptr.get() << "\n";
    }
    {
        std::string s1 = "hello";
        std::string s2 = std::move(s1);
        // s1 is 'moved-from': valid, but unspecified.
        // May be empty, may be "hello", may be in a random state.
        std::cout << "|" << s1 << "| |" << s2 << "|\n";
        assert(s1.empty());  // FIXME: may fail
        s1 = "ok";

        /*
        struct string {
            char buf_small[10];
            char *buf_big = nullptr;
            void move_from(string &other) {
                if (other.buf_big) {
                    buf_big = other.buf_big;
                    other.buf_big = nullptr;
                } else {
                    buf_small[0] = other.buf_small[0];
                    buf_small[1] = other.buf_small[1];
                    // ...
                }
            }
        };
        */
    }
}  
```

Тесно связано с: функции (как передавать параметры), жизнь объектов.
