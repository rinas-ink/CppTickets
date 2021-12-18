## Билет 22. Статические члены класса
Обычные члены класса относятся к конкретному объекту. Статические члены же относятся ко всему классу. 
```c++
struct Foo {
    Foo() {
        Foo::objectCount++;
    }
    static int objectCount;
};

int Foo::objectCount = 0; // exactly one definition in exactly one TU!

int main() {
    std::cout << Foo::objectCount << '\n';  // 0
    Foo a1;
    Foo a2;
    std::cout << Foo::objectCount << '\n';  // 2
}
```
### Статические поля
Обратите внимание, что в примеры выше поле `objectCount` **объявлено и определено раздельно**.

Почему так? Потому что непонятно, где нужно выделить память на это поле. Где мы определили, там и выделится.

Кроме того определение статического поля должно быть **одно и только в одной единице трансляции**. ODR.

#### Слово `inline`
Если хочется всё-таки сразу объявить поле внутри класса, то можно добавить ключевое слово `inline`. Тогда объявление вне класса не нужно. 

С классическим смыслом `inline` ODR-выключателя запутанно. Экспериментально выявил, что нельзя несколько раз объявлять, даже в разных единицах трансляции, т.е. ODR не выключился... Хотя вроде по cppreference должно быть можно, ибо external linkage... Почитайте [cppreference](https://en.cppreference.com/w/cpp/language/inline#:~:text=An%20inline%20function%20or%20variable%20(since%20C%2B%2B17)%20with,It%20has%20the%20same%20address%20in%20every%20translation%20unit).
<!-- FIXME: --->

```c++
struct S {
    int n;                    // defines S::n
    static int i;             // declares, but doesn't define S::i
    inline static int x;      // defines S::x
    inline static int y = 1;  // defines S::y
};

int S::i = 3;                 // defines S::i
```

### Обращение к статическим членам
Предполагаемое обращение к статическим члена это `Foo::member`, но для удобства можно и по-другому:
```c++
struct Foo {
    static inline int x = 1;
    static void bar() {}
    void baz() {
        Foo::x;
        Foo::bar();

        x;
        bar();

        this->x;
        this->bar();
    }
};

int main() {
    Foo::x;
    Foo::bar();
    
    Foo obj;
    obj.x;
    obj.bar();
}
```
На статические члены также ожидаемым образом влияют `public/protected/private`. Но объявить статическое поле снаружи тоже можно и доступ к внутренним .
```c++
struct Foo {
private:
    static inline int x = 2;
    static int y;
};

int Foo::y = x + 2;  // ok!

// int a = Foo::y  // can't access private!
``` 

### Порядок инициализации и удаления статических полей
Статические поля имеют static storage duration (см. [билет 15](https://github.com/khbminus/CppTickets/blob/master/tickets/ticket15.md)), поэтому внутри единицы трансляции гарантируется, что инициализация в том порядке, как она записана в файле (кроме некоторых `const`! так как они инициализируются через *const initialization* обычно сразу на этапе компиляции).

Порядок удаления неочевиден (под разными компиляторами даже по-разному).

Больше про это можно прочитать в [билете 33](https://github.com/khbminus/CppTickets/blob/master/tickets/ticket33.md) про static initialization fiasco.

### Статические константы
Если поле сделать `integral` (типа, которое ~почти int: `int`, `char`, `long`, etc.) поле константным, то его можно сразу инициализировать через initializer:

```c++
struct Foo
{
    const static int n = 1;
    const static int m{2};  // Since C++11
    const static int k;
};
const int Foo::k = 3;
```

#### Беды с константами
Кроме этого могут происходить беды с указателями на константы, например:

```c++
// foo.h
struct Foo {
    static const int N = 60; // объявление
}
// fst.cpp
... Foo::N ... // ОК
... &Foo::N ... // может быть undefined reference на этапе линковки
// snd.cpp
... Foo::N ... // ОК
... &Foo::N ... // может быть undefined reference на этапе линковки
```

В чем проблема? У константы должно быть и объявление, и должно быть определение ровно в одной единицы трансляции. Для решения проблемы добавляем определение констант.

```c++
// foo.h
struct Foo {
    static const int N = 60; // объявление
}
// fst.cpp
const int Foo::N; // инициализация и слово `static` не нужны
... Foo::N ... // ОК
... &Foo::N ... // ОК
// snd.cpp
... Foo::N ... // ОК
... &Foo::N ... // ОК
```
Если не писать inline, то необходимо эту константу определить ровно в одном cpp файле, иначе ошибка компиляции (пример выше). Если в двух и более файлах написали определение, то ошибка компиляции.

Если пишем inline, то больше ничего определять не нужно.

```c++
// foo.h
struct Foo {
static inline const int N = 60;
}
// fst.cpp
... Foo::N ... // ОК
... &Foo::N ... // ОК
// snd.cpp
... Foo::N ... // ОК
... &Foo::N ... // ОК
```

Ровно так следует писать константы по мнению Егора.
### Статические методы, отличия от свободных функций и друзей
В статических методах, как можно догадаться, не видно `this`. Обращаться можно только к статическим членам.

Все статические методы сразу `inline`.

* От свободных функций отличаются уровнем доступа: у `static` полный доступ.
* От друзей отличаются синтаксисом. Теоретически можно написать то же самое, но обычно хочется вложить другой смысл.

### Паттерн: статический метод как конструктор с именем
Допустим, хочется класс прямоугольник. Хочется уметь создавать его по координатам левого-нижнего и правого-верхнего концов: `x1`, `y1`, `x2`, `y2`, а также по координате левого-нижнего конца, ширине и высоте: `x1`, `y1`, `width`, `height`.

И то, и то - четыре int, значит signature одинаковый, поэтому компилятор не поймёт, какой вы хотите конструктор.

Вместо этого можно сделать два статических метода вместо конструктора:
```c++
struct Rectangle {
    Rectangle(...) {...}

    static Rectangle makeFromPoints(int x1, int y1, int x2, int y2) {
        return Rectangle(...);
    }

    static Rectangle makeFromWidthAndHeight(int x1, int y1, int width, int height) {
        return Rectangle(...);
    }
};
```

### Полезное
* https://en.cppreference.com/w/cpp/language/inline
* https://en.cppreference.com/w/cpp/language/definition

### Неочевидные источники:
* https://github.com/vladnosiv/hse-spb-conspects-2020/blob/master/C%2B%2B/all-tickets.md#%D0%B1%D0%B8%D0%BB%D0%B5%D1%82-01-%D0%B4%D0%B5%D1%82%D0%B0%D0%BB%D0%B8-%D0%BA%D0%BB%D0%B0%D1%81%D1%81%D0%BE%D0%B2
* https://olympiads.ru/zaoch/2018-19/lang_docs/cppreference.com/reference/en/cpp/language/static.html