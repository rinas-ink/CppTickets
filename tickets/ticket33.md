## Билет 33. Static initialization order fiasco
### О чём это
Рассмотрим программу, состоящую из нескольких единиц трансляции: `a.cpp` и `b.cpp`. Пусть в `a.cpp` и `b.cpp` создаются объекты с статическим временем жизни (см. [билет 15](https://github.com/khbminus/CppTickets/blob/master/tickets/ticket15.md#static-storage-duration)), например глобальные переменные. В таком случае порядок инициализации этих переменных зависит от порядка линковки этих единиц трансляции. Это может привести к проблемам.
<!-- TODO: make references to other tickets uniform --->
### Создание и уничтожение объектов со статическим временем жизни
В целом, объекты с static storage duration инициализируются при запуске программы (в каком-то порядке) и удаляются по завершению программы (в неочевидном порядке<sup>[1](https://www.youtube.com/watch?v=XdrSzs04HKU&list=PL8a-dtqmQc8obAqSKqGkau8qiafPRCxV7&t=4806s)</sup> <sup>[2](https://stackoverflow.com/questions/31443437/why-is-the-order-of-destruction-of-these-function-local-static-objects-not-the-i)</sup>, под разными компиляторами по-разному). Также стандарт C++ гарантирует, что все static storage duration объекты внутри одной единицы трансляции будут проинициализированы по порядку<sup>[3](https://en.cppreference.com/w/cpp/language/initialization#:~:text=initialization%20of%20these%20variables%20is%20always%20sequenced%20in%20exact%20order%20their%20definitions%20appear%20in%20the%20source%20code.)</sup>, но нет никаких гарантий про порядок между ними! 
#### У кого static storage duration
Есть деление на два типа<sup>[4](https://en.cppreference.com/w/cpp/language/storage_duration#:~:text=static%20or%20extern.-,See%20Non%2Dlocal%20variables%20and%20Static%20local%20variables%20for%20details%20on%20initialization%20of%20objects%20with%20this%20storage%20duration.,-thread%20storage%20duration)</sup>: *non-local* переменные (в сущности, глобальные или статические поля класса) и *static local* переменные (статические локальные).
##### Non-local
К примеру, обычные глобальные переменные.
```c++
int x = 1;

void foo() {
    x++;
}

int main() {
    std::cout << x << '\n';  // 1
    foo();
    std::cout << x << '\n';  // 2
    x += 2;
    std::cout << x << '\n';  // 4
}
```
Как правило, они инициализируются во время запуска программы. Однако в случае non-local переменных, если применимо *constant initialization*<sup>[5](https://en.cppreference.com/w/cpp/language/constant_initialization)</sup>, то компилятор может (но не обязан, хотя обычно так и есть) создать объект сразу на этапе компиляции! Таким образом, объект будет встроен в .exe файл, из-за чего он может раздуться.

Если же constant initialization не применимо, то сначала используется *Zero initialization*<sup>[6](https://en.cppreference.com/w/cpp/language/zero_initialization)</sup>:
```c++
struct A {
    int a,b,c;
};
 
double f[3]; // zero-initialized to three 0.0's
int* p; // zero-initialized to null pointer value (even if the value is not integral 0)
std::string s; // zero-initialized to indeterminate value
               // then default-initialized to "" by the std::string default constructor
int main(int argc, char*[])
{
    delete p; // safe to delete a null pointer
    static int n = argc; // zero-initialized to 0 then copy-initialized to argc
    std::cout << "n = " << n << '\n';
    A a = A(); // the effect is same as: A a{}; or A a = {};
    std::cout << "a = {" << a.a << ' ' << a.b << ' ' << a.c << "}\n";
}
```

После *Zero initialization* идёт *Dynamic initialization*<sup>[7](https://en.cppreference.com/w/cpp/language/initialization#Dynamic_initialization)</sup>, собственно присвоение значений.

Компилятор имеет право сделать *Early Dynamic initialization*<sup>[8](https://en.cppreference.com/w/cpp/language/initialization#Early_dynamic_initialization)</sup>, обычно на этапе компиляции, если он видит, что объект не меняет другие объекты и не зависит от других не *early dynamic initialized*.
```c++
inline double fd() { return 1.0; }
extern double d1;
double d2 = d1;   // unspecified:
                  // dynamically initialized to 0.0 if d1 is dynamically initialized, or
                  // dynamically initialized to 1.0 if d1 is statically initialized, or
                  // statically initialized to 0.0 (because that would be its value
                  // if both variables were dynamically initialized)
double d1 = fd(); // may be initialized statically or dynamically to 1.0
```
Тут вообще много всего интересного и запутанного. Если есть время, посмотрите [cppreference](https://en.cppreference.com/w/cpp/language/initialization).
##### Static local
Статические локальные переменные можно создавать внутри функций, тогда они будут доступны каждый раз, когда вызывается эта функция.
```c++
int foo(int addition) {
    static int x = 1;
    x += addition;
    return x;
}

int main() {
    std::cout << x << '\n';  // 1
    foo(1);
    std::cout << x << '\n';  // 2
    foo(2);
    std::cout << x << '\n';  // 4
}
```
Инициализируется такой объект единожды первый раз, когда он используется в функции.
### Пример static initialization order fiasco
Из-за того, что порядок инициализации зависит от порядка линковки, можно наткнуться на проблемы, если один объект при своей инициализации (обычно в конструкторе) использует другой объект, который сам ещё не создался.

Это и называется static initialization order fiasco (SIOF).
#### Пример с некорректным порядком
`a.h`
```c++
#ifndef HSE_CPP_EXAM_A_H
#define HSE_CPP_EXAM_A_H

#include <vector>

struct Counter {
    explicit Counter (int init_count) : count(init_count) {}

    int getID() {
        return count++;
    }
    int count;
};

extern Counter globalCounter;

#endif //HSE_CPP_EXAM_A_H 
```

`a.cpp`
```c++
#include <vector>
#include "a.h"

Counter globalCounter(100);
```
`main.cpp`
```c++
#include <iostream>
#include "a.h"

int someId = globalCounter.getID();

int main() {
    std::cout << someId << '\n';  // either 100 or uninitialized
}
```
Если скомпилировать файлы в порядке `g++ a.cpp main.cpp -o siof`, то всё будет хорошо. Если же `g++ main.cpp a.cpp -o siof`, то `someId` не сможет воспользоваться `globalCounter`.
#### Пример, где не существует корректного порядка
`a.h`
```c++
#ifndef HSE_CPP_EXAM_A_H
#define HSE_CPP_EXAM_A_H

#include <utility>
#include <vector>
#include <iostream>

struct Counter {
    explicit Counter (int init_count, std::string name) : count(init_count), name(std::move(name)) {}

    int getID() {
        return count++;
    }

    int count;
    std::string name;
};

extern Counter globalCounter;

#endif //HSE_CPP_EXAM_A_H
```
`a.cpp`
```c++
#include "a.h"
#include "b.h"

Counter globalCounter(100, globalNameGiver.getName());
```
`b.h`
```c++
#ifndef HSE_CPP_EXAM_B_H
#define HSE_CPP_EXAM_B_H

struct NameGiver {
    explicit NameGiver(int id) : id(id) {}

    std::string getName() {
        return "obj" + std::to_string(id);
    }

    int id;
};

extern NameGiver globalNameGiver;

#endif //HSE_CPP_EXAM_B_H
```
`b.cpp`
```c++
#include "a.h"
#include "b.h"

NameGiver globalNameGiver(globalCounter.getID());
```
`main.cpp`
```c++
#include <iostream>
#include "a.h"
#include "b.h"

static int someId = globalNameGiver.id;
static std::string someName = globalCounter.name;

int main() {
    std::cout << someId << '\n';  // ???
    std::cout << someName << '\n';  // ???
}
```
`globalCounter` нужен для создания `globalNameGiver`, а `globalNameGiver` нужен для создания `globalCounter`... Вопрос о курице и яйце без решения.
#### Пример, где возникает UB только через std::vector
Давайте в модуль положим вектор, значение которого инициализируем только в `a.cpp`.

`a.h`
```c++
#ifndef HSE_CPP_EXAM_A_H
#define HSE_CPP_EXAM_A_H

#include <vector>

struct Foo {
    static std::vector<int> a;
};
// or use extern instead of Foo

#endif //HSE_CPP_EXAM_A_H
```
`a.cpp`
```c++
#include <vector>
#include "a.h"

std::vector<int> Foo::a{1, 2, 3};
```
`main.cpp`
```c++
#include <iostream>
#include "a.h"

int first_of_a = Foo::a[0];

int main() {
    std::cout << first_of_a << '\n';  // either 1, or UB, as it's out-of-bounds
}
```
Если сначала инициализируются объекты из `main.cpp`, то `first_of_a` должно взять значение неинициализированного вектора, получая UB.
### Решение проблемы
Чтобы избежать этой проблемы, можно воспользоваться идиомой 'construct on first use': вместо `non-local` переменных, будем использовать `static local`, чтобы они гарантированно создались, когда мы ими воспользовались.

Например, для прошлого примера:

`a.h`
```c++
#ifndef HSE_CPP_EXAM_A_H
#define HSE_CPP_EXAM_A_H

#include <vector>

struct Foo {
    static std::vector<int>& getVector();
};

#endif //HSE_CPP_EXAM_A_H
```
`a.cpp`
```c++
#include <vector>
#include "a.h"

std::vector<int>& Foo::getVector() {
    static std::vector<int> a{1, 2, 3};
    return a;
}
```
`main.cpp`
```c++
#include <iostream>
#include "a.h"

int first_of_a = Foo::getVector()[0];

int main() {
    std::cout << first_of_a << '\n';  // now OK!
}
```
#### Сравнение с автоматическим временем жизни и динамическим
* В отличие от автоматического времени жизни, один и тот же объект может использоваться в разных единицах трансляции.
* В отличие от динамического времени жизни, такой способ безопаснее, потому что гарантируется, что объект живой.

### Замечания
* Циклическое SIOF починить не получится никак, так как оно циклическое...
* Такие же приколы могут быть с уничтожением объектов: может удалится объект, нужный другому в деструкторе. Такое можно решать, например, вообще никогда не удаляя объект:
```c++
Foo& getFoo() {
    static auto* ptr = new Foo();  // never destructs!
    return *ptr;
}
```
* `cin`, `cout` тоже глобальные переменные, поэтому если вы используете их в конструкторе, то теоретически могут быть такие же проблемы. Однако, начиная с C++11, гарантируется, что `cout`, `cin` и прочие создадутся раньше остальных объектов с статическим временем жизни. **Только в случае если `#include <iostream>` идёт до `#include` файла с объявлением**. Об этом можно поподробнее прочитать в [ubbook](https://github.com/Nekrolm/ubbook/blob/master/runtime/static_initialization_order_fiasco.md#initialization-order-fiasco-%D0%B8-%D0%BD%D0%B5%D0%B8%D1%81%D0%BF%D0%BE%D0%BB%D1%8C%D0%B7%D1%83%D0%B5%D0%BC%D1%8B%D0%B5-%D0%B7%D0%B0%D0%B3%D0%BE%D0%BB%D0%BE%D0%B2%D0%BA%D0%B8).

### Полезные ссылки
* https://isocpp.org/wiki/faq/ctors#static-init-order
* https://github.com/Nekrolm/ubbook/blob/master/runtime/static_initialization_order_fiasco.md
* https://en.cppreference.com/w/cpp/language/initialization