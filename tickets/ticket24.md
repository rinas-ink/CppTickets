## Билет 24. Преобразования

Преобразование одного типа в другой.  
Бывают явные(explicit) и неявные(implicit).  
Явные — программист чётко прописывает, что хочет преобразовать тип:

```C++
double a = 123.4;
int b = static_cast<int>(a); // кастуем a к int
```

```C++
struct foo{ foo(int){} }
void bar(foo){}
int main{
    int k = 2;
    bar(foo(k)); // создаём объект foo от k и передаём в функцию
}
```

Неявные — умняш компилятор сам догадывается, что во что нужно преобразовать.

```C++
double a = 123.4;
int b = a; // компилятор сам понял, что хотим привести double к int
```

```C++
struct foo{ foo(int){} };
void bar(foo){}
int main(){
    int k = 2;
    bar(k); // компилятор догадался создать foo от int k и передать его в функцию bar
}
```

### Implicit

```C++
#include <iostream>

struct ratio {
    int num;
    int denom;
    
    // конструктор без параметров
    ratio() {
        num = 0;
        denom = 1;
        std::cout << "Default constructor\n";
    }
    
    // конструктор с параметром типа int
    ratio(int value) {
        num = value;
        denom = 1;
        std::cout << "ratio(int)\n";
    }
};

// функция ожидает на вход константную ссылку на объект типа ratio
void println(const ratio &r) {
    std::cout << r.num << " " << r.denom << "\n";
}

ratio generate_ratio() {
    return 123; // превращается в return ratio(123);
}

int main() {
    ratio r = 10; // вызывается конструктор ratio(int)
    println(r); 
    println(10); // println(ratio(10));
    println(generate_ratio());
}
```

В коде выше 3 раза происходит неявное преобразование.  
Для `ratio r = 10;` компилятор выполнил то же самое, что выполнил бы для `ratio r = ratio(10);`, то есть произошло
неявное преобразование int — 10 к ratio.  
`println(10)` здесь — это `println(ratio(10))`.  
Наконец

```C++
ratio generate_ratio() {
    return 123;
}
```

есть на самом деле

```C++
ratio generate_ratio() {
    return ratio(123);
}
```

Во всех трёх случаях компилятор пытается найти конструктор, в который может передать int и неявно его использовать.

#### Пытливые умы спросят

Что если сделать `ratio r = 1.4;`?  
В этом случае компилятор запустит цепочку преобразований double->int->ratio. Подробнее об этом в последнем параграфе.

### Почему неявные преобразования — не всегда хорошо?

```C++
void print_vector([[maybe_unused]] const std::vector<int> &vec) {
}

int main() {
    [[maybe_unused]] std::vector<int> v1(10); // Норм
    [[maybe_unused]] std::vector<int> v2 = 10; // Не норм
    print_vector(10); // Почему, а главное зачем?
}
```

Существует конструктор vector(int n), который создаёт вектор из n дефолтных значений.  
Но тогда почему бы не вызвать неявно `[[maybe_unused]] std::vector<int> v2 = 10;`?  
Потому что выглядит ~~всрато~~ не интуитивно. Вполне разумно было бы ожидать от такой конструкции получить вектор {10,
}.  
Совсем странно выглядит `print_vector(10);`
Вроде хотим вывести вектор, а аргумент — число. Непорядок.  
Благо в stl умные дяди позаботились и _предусмотрели_, поэтому эта штука не скомпилируется, но если бы вместо
стандартного vector мы бы использовали самописную структуру

```C++
struct my_vector {
    my_vector() {}
    my_vector(int) {}
};
```

всё скомпилировалось бы.

### Чтобы не напороться на такую проблему существует Explicit.

Помечаем конструктор словом explicit и запрещаем неявные преобразования.

```C++
struct my_vector {
    my_vector() {}
    explicit my_vector(int) {}
};
```

Теперь можем только явно вызвать конструктор от int.

```C++
    [[maybe_unused]] my_vector v1(10); // Компилится
    [[maybe_unused]] my_vector v2 = static_cast<my_vector>(10); // Компилится
    // [[maybe_unused]] my_vector v2 = 10; // Не компилится
    // print_vector(10); // Не компилится
```

`static_cast<T>` — явное преобразование к типу T.

#### Explicit можно использовать для конструкторов с несколькими параметрами или без параметров

Например, хотим запретить неявно создавать объект при помощи `{}`.  
Это полезно, потому что зачастую `Object{a, b, c}` интуитивно понятнее и логичнее, чем `{a, b, c}` 

```C++
struct Foo {
    explicit Foo() {}

    explicit Foo(int, std::string) {}
};

void call(const Foo &) {
}

Foo ret() {
    return Foo{}; // OK
    // return {}; // BAD

    return Foo{10, "hello"}; // OK
    // return {10, "hello"}; // BAD
}

int main() {
    [[maybe_unused]] Foo f1{10, "hello"}; // OK
    // [[maybe_unused]] Foo f2 = {10, "hello"}; // BAD

    [[maybe_unused]] Foo f3{}; // OK

    [[maybe_unused]] Foo f4 = Foo{}; // OK
//    [[maybe_unused]] Foo f4 = {}; // BAD

    call(Foo{10, "hello"}); // OK
    // call({10, "hello"}); // BAD

    call(Foo{}); // OK
    // call({}); // BAD
    ret();
}
```

#### Можно ли запретить только некоторые неявные преобразования?

> По умолчанию нельзя, но в конце 4 модуля можно будет разрешить преобразование в произвольный тип, а потом вызывать ошибку компиляции, если он не совпал с желаемым. Но там больно.
> —Егор

### Операторы преобразования

#### Проблема

```C++
struct ratio {
    int num;
    int denom;
};

int main() {
    ratio r{3, 4};
    double x = r; // Нельзя
}
```

Не можем преобразовать объект нашего класса в double, потому что в double нет конструктора от ratio. Можно было бы
дописать, будь он нашим классом, но double, во-первых, не наш, во-вторых, не класс.

#### Решение

```C++
struct ratio {
    int num;
    int denom;

    operator double() const {
        std::cout << "operator double()\n";
        return num * 1.0 / denom;
    }
};
```

Дописали `operator double() const{}` теперь наш ratio умеет конвертироваться в double.

#### Приятный побочный эффект

Начинает работать

```C++
ratio r{3, 4};
std::cout << r << "\n";
```

Потому что `std::cout` умеет работать от double, а ratio теперь умеет неявно преобразовываться в double.  
То есть на самом деле код выше отработает, как
```C++
ratio r{3, 4};
std::cout << r.operator double() << "\n"; // явное преобразование к double
std::cout << static_cast<double>(r) << "\n"; // то же самое
```

### Можно пометить оператор explicit

```C++
struct ratio {
    int num;
    int denom;

    epxlicit operator double() const {
        std::cout << "operator double()\n";
        return num * 1.0 / denom;
    }
};
```

Например, решили, что не хотим неявно преобразовывать в double, чтобы не потерять точность.

```C++
ratio r{3, 4};
double x = r; // не скомпилируется, потому что запретили неявное преобразование
double x = static_cast<double>(r); // явное преобразование к double — работает
std::cout << x << "\n";
std::cout << static_cast<double>(r) << "\n"; // тоже явное, тоже работает
```

### Ошибки из-за неоднозначности

```C++
struct Foo {
    // оператор преобразования Foo к Bar
    operator Bar() {
        return Bar{};
    }
};

struct Bar {
    Bar() {}
    // конструктор Bar от Foo
    Bar(Foo /*arg*/) {}
};

int main() {
    Foo f;
    Bar b = f;  // ambiguous
}
```

Foo умеет преобразовываться в Bar, но и Bar умеет создаваться от Foo.  
То есть имеем два способа преобразовать Foo к Bar, между которыми компилятор не может выбрать, поэтому не компилирует
вовсе.
#### Можно починить
Для этого пометим один из методов explicit
```C++
struct Bar;

struct Foo {
    operator Bar();
};

struct Bar {
    Bar() {}

    explicit Bar(Foo /*arg*/) {
        std::cout << "Bar(Foo)";
    }
};

Foo::operator Bar() {
    std::cout << "Foo::operator Bar()";
    return Bar{};
}

int main() {
    Foo f;
    Bar b = f;
}
```
Теперь неявно может вызваться только `Foo::operator Bar()`, это и произойдёт.

### Возвращаясь к неявным преобразованиям

#### Порядок преобразований

Последовательность преобразований состоит из:
<ol>
  <li>0 или 1 стандартная последовательность преобразований.</li>
  <li>0 или 1 пользовательское преобразование.</li>
  <li>0 или 1(только если было выполнено пользовательское преобразование) стандартная последовательность преобразований.</li>
</ol>

Стандартная последовательность преобразований — например, преобразование между численными типами. Подробнее про это
можно почитать [здесь](https://en.cppreference.com/w/cpp/language/implicit_conversion)

##### Пример

```C++
struct Foo {
    Foo(int) {}
};

struct Bar {
    Bar(const Foo&) {}
};

void call_with_bar(const Bar&) {
}

int main() {
    // На стандартные преобразования тип 'Bar -> const Bar' забиваем, но вообще они есть.
    call_with_bar(Bar(Foo(10)));  // все преобразования явные
    call_with_bar(Bar(Foo(10LL)));  // стандартное неявное преобразование long long -> int
    call_with_bar(Foo(10));  // пользовательское неявное преобразование Foo -> Bar
    call_with_bar(Bar(10));  // пользовательское неявное преобразование int -> Foo
    call_with_bar(Bar(10LL));  // стандартное неявное преобразование long long -> int + пользовательское неявное преобразование int -> Bar
    // call_with_bar(10);  // два неявных пользовательское преобразование: int -> Foo -> Bar
}
```

Строчка `call_with_bar(10);` не скомпилируется, потому что ей требуется два пользовательских преобразования, а
допускается только 0 или 1.

##### Зачем такие ограничения?

Потому что в момент вызова `call_with_bar(10)` совершенно неясно почему и откуда должна взяться промежуточная структура
Foo, а ведь она может быть и не одна.

```C++
struct Foo {
    Foo(int) {}
};

struct Baz {
    Baz(int) {}
};

struct Bar {
    Bar(const Foo&) {}
    Bar(const Baz&) {}
};

void call_with_bar(const Bar&) {
}
int main() {
    call_with_bar(10);
}
```

Чтобы это заработало, пришлось бы перебирать все промежуточные вершины преобразований(int -> Foo -> Bar или int -> Baz
-> Bar), а это долго, сложно, и стреляло бы по ногам, поэтому запретили.

### std::string — обычный пользовательский тип

Он определен в стандартной библиотеке, но с точки зрения языка ничем не примечателен. Отсюда следующее:

```C++
#include <string>

struct Foo {
    Foo(const std::string&) {}
};

void call_with_foo(const Foo&) {
}

int main() {
    call_with_foo(Foo(std::string("hello"))); // все преобразования явные
    call_with_foo(Foo("hello"));  // неявное пользовательское преобразование const char[6] -> std::string
    call_with_foo(std::string("hello")); // неявное пользовательское преобразование std::string -> Foo
    // call_with_foo("hello");  // требуется 2 неявных преобразования

    {
        using namespace std::literals; // используем стандартные литералы
        call_with_foo("hello"s);  // "hello"s — это std::string, 
                                  // поэтому здесь 1 неявное преобразование string -> Foo
    }
}
```

### Оператор bool

Позволяет преобразовывать объект к bool, например

```C++
while(std::cin >> x){}
whlie((std::cin >> x).operator bool()){}
```

это две одинаковых строчки.

#### Проблема

С точки зрения языка bool — это число и его легко неявно преобразовать к int. Причём такое преобразование является
стандартным, а не пользовательским.  
Из-за этого можно сделать вот так:

```C++
#include <iostream>

struct Foo {
    operator int() {
        return 10;
    }
};

struct Bar {
    operator bool() {
        return true;
    }
};

int main() {
    [[maybe_unused]] Foo f;
    [[maybe_unused]] Bar b;
    [[maybe_unused]] bool x1 = f;  
    [[maybe_unused]] bool x2 = b;  

    if (f) {}
    if (b) {}

    for (; f;) {}
    for (; b;) {}

    while (f) {}
    while (b) {}

    [[maybe_unused]] int x3 = 10 + f;
    [[maybe_unused]] int x4 = 10 + b;
}
```

Foo и Bar могут быть какими-то сложными структурами, а мы используем их как bool или int.  
Использовать Bar как bool — нормально, потому что у него для этой цели определён operator, но очевидно не ожидалось, что
кто-то будет использовать этот класс для арифметики.  
Аналогично и Foo предполагалось преобразовывать только к int, но никак не к bool.  
В C++03 с этим боролись при помощи [safe bool idiom](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Safe_bool).

#### Начиная с C++11 решение приятное и лаконичное.

Просто дописываем к операторам explicit, чтобы запретить неявные преобразования.

```C++
#include <iostream>

struct Foo {
    explicit operator int() {
        return 10;
    }
};

struct Bar {
    explicit operator bool() {
        return true;
    }
};

int main() {
    [[maybe_unused]] Foo f;
    [[maybe_unused]] Bar b;
    // [[maybe_unused]] bool x1 = f;
    // [[maybe_unused]] bool x2 = b;

    // if (f) {}  // неявное преобразование int -> bool запрещено
    if (b) {}  // преобразовали к bool

    // for (; f;) {}
    for (; b;) {}

    // while (f) {}
    while (b) {}
    
    [[maybe_unused]] int x3 = 10 + static_cast<int>(f); // явно преобразовали к int
    // [[maybe_unused]] int x3 = 10 + f; // не срабоатет, потому что неявное
    // [[maybe_unused]] int x4 = 10 + b; 
}
```

Теперь Bar можем использовать только как bool, а Foo только как int и только при явном преобразовании.  
Важно заметить, что по стандарту языка внутри конструкций `if, for, while` преобразование к bool является **явным**.
