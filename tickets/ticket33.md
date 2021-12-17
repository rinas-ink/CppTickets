## Static initialization order fiasco
### О чём это
Рассмотрим программу, состоящую из нескольких единиц трансляции: `a.cpp` и `b.cpp`. Пусть в `a.cpp` и `b.cpp` создаются объекты с статическим временем жизни (см. [билет 15](https://github.com/khbminus/CppTickets/blob/master/tickets/ticket15.md#static-storage-duration)), например глобальные переменные. В таком случае порядок инициализации этих переменных зависит от порядка линковки этих единиц трансляции. Это может привести к проблемам.
<!-- TODO: make references to other tickets uniform --->
### Создание и уничтожение объектов со статическим временем жизни
В целом, объекты с static storage duration инициализируются при запуске программы (в каком-то порядке) и удаляются по завершению программы (гарантированно в обратном порядке). Также стандарт C++ гарантирует, что все static storage duration объекты внутри одной единицы трансляции будут проинициализированы по порядку, но нет никаких гарантий про порядок между ними! 
#### У кого static storage duration
Есть деление на два типа<sup>[2](https://en.cppreference.com/w/cpp/language/storage_duration#:~:text=static%20or%20extern.-,See%20Non%2Dlocal%20variables%20and%20Static%20local%20variables%20for%20details%20on%20initialization%20of%20objects%20with%20this%20storage%20duration.,-thread%20storage%20duration)</sup>: *non-local* переменные (в сущности, глобальные или статические поля класса) и *static local* переменные (статические локальные).
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
Как правило, они инициализируются во время запуска программы. Однако в случае non-local переменных, если применимо *constant initialization*<sup>[3](https://en.cppreference.com/w/cpp/language/constant_initialization)</sup>, то компилятор может (но не обязан, хотя обычно так и есть) создать объект сразу на этапе компиляции! Таким образом, объект будет встроен в .exe файл, из-за чего он может раздуться.
##### Static local
Статические локальные переменные можно создавать внутри функций, тогда они будут доступны каждый раз, когда вызывается эта функция.
```c++
int foo() {
    static int x = 1;
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

### Полезные ссылки
* https://isocpp.org/wiki/faq/ctors#static-init-order
* https://en.cppreference.com/w/cpp/language/initialization#Non-local_variables