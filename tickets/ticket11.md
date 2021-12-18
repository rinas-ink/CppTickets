## Билет 11. Объявление объектов внутри одного файла

* ### Объявление и определение: функции, класса

Компилятор - штука старая, совместимая с Си, поэтому компиляция идет по файлу сверху вниз. (в общем это правда, НО в
структурках сделали, как у нормальных людей, тк тут на надо было в обратную совместимость)

**Declaration**(forward) - объявление для того, чтобы показать компилятору, что существует такая функция/класс, чтобы
можно было использовать раньше, чем мы написали реализацию.

**Definition** - определение, прописываем реализацию

В этом примере можно просто поменять порядок и не нужно будет объявлять

```c++
    void foo(); // declaration
    // еслли без этого, мы хз че такое foo в print() ниже
    
    struct Iam{ // definition
        void print(){
            cout << "I am";
            foo();
        }
    };
    
    void foo() { // definition
        cout << "tired";
    }
```

* ### Взаимная рекурсия для: функций, классов, методов внутри одного класса, методов между классами (A::foo() возвращает B и наоборот).

    * Функции

      Могут быть кеки, вида одна фукнция запускает дургую и наоборот, тогда без forward declaration мы не скомпилимся.

        ```c++
        void bar(int n);  // declaration, объявление
        // void bar(int = 10);  // declaration, объявление
        // Default arguments are better be specified in declaration.
        
        void foo(int n) {  // definition, определение
            std::cout << "foo " << n << "\n";
            bar(n - 1);
        }
        
        void bar(int n) {  // definition, определение
            std::cout << "bar " << n << "\n";
            if (n == 0) {
                return;
            }
            foo(n - 1);
        }
        
        int main() {
            bar(10);
        }
        ```
    * Методы внутри класса

      Как я писал выше, в структурках сделали как у белых людей, те мы видим все, что лежит у нас в структуре:
        ```c++
        struct Foo{
            void foo() {
                cout << 1;
                bar();
            }
            void bar() {
                cout << 2;
            }
            struct Bar{};
        };
        
        int main(){
            Foo::Bar b; // 
        }
        ```
    * Классы

      Ну кста методы можно просто объявить в структурке, а объявлять в наруже(через Foo::)

      Можем хранить только указатели, ветора... кароче штуки, котрым пофиг на размер того, что ты им подсунул(вектора
      там чет себе на куче делает, указатель - и в Африке указатель)
        ```c++
        struct Bar;

        struct Foo {
            // Bar b - так нельзя, тк мы не знаем сколько байтов занимает Bar, да и вообще бесконечная глубина
            Bar *b;
            std::vector<Bar> bs;
        };
        
        struct Bar {
            Foo f;
        };
        ```

      А теперь бахнем взаимную рекурсию:

      Было(не работает шо пипец):
        ```c++
            struct Foo {
                operator Bar() {
                    return Bar{};
                }
            };

            struct Bar {
                Bar() {}
                Bar(Foo /*arg*/) {}
            };
            
            int main() {
                Foo f;
                Bar b = f;  // ambiguous
            }
        ```
      Сначала мы не знаем, что Bar() - тип, для этого fwd declaration.  
      Получилось, что Bar - incomplete, мы не знаем что за мусор в Bar, когда пытаемся сделать оператор преобразования к
      Bar. Ну тогда просто реализуем этот метод после того, как узнаем про Bar(в Питере - пить!). Получили, что и
      хотели, что преобразование типов(в мэйне) неоднозначно(надо еще explicit куда-нибдь бахнуть и норм)         
      Стало:

        ```c++
            
            struct Bar;  // incomplete type

            struct Foo {
                operator Bar();
            };

            struct Bar {
                Bar() {}
                Bar(Foo /*arg*/) {}
            };
            
            Foo::operator Bar() {
                return Bar{};
            }
            
            int main() {
            Foo f;
            Bar b = f;  // ambiguous
            }
        ```
* ## Incomplete type: как объявить, что можно сделать с неполным типом.

Incomplete - это тип, который описывает идентификатор, но не содержит информацию, необходимую для определения размера
идентификатора. Типо мы не знаем его размер, тк пока только объявили, но можем указатели тыкнуть или в векторочек
положить, ссылками на него побаловаться итп
смотри [сюда](https://github.com/hse-spb-2021-cpp/lectures/tree/master/06-211006/09-incomplete). Мы можем просто
объявить не полный тип, пожонглировать его ссылками, не заглядывая под капот чего там происходит.

* ## Namespaces

Если есть две одинковые по назавнию и параметрам функции, переменные итп, которые делают что-то разное, то мы можем из
обернуть в пространства имен, чтобы уметь обращаться к нужной.

```c++
void foo(){
    cout << "global foo";
};
void  some_glob(){
    cout << "global kek";
};
namespace ns1{
void bar(){
    cout << "В Питере - пить!";
}
void foo(){
    cout << "foo1";
    bar();
    some_glob();
};

}
namespace ns2{
void foo(){
    cout << "foo2";
};
}
int main() {
    ns1::foo();
    ns2::foo();
    ::foo() // то же самое, что foo() - в глабольном namespace
}
```
Кста в нэймспейсах важен порядок в котором мы объявляем функции(те не как в классах), идем сверзу вниз.

Можем сделать их вложенными(читаем комментики):

```c++
void foo() {
    std::cout << "foo global\n";
}

void some_global() {
    std::cout << "some_global\n";
}

namespace ns1 {
void bar() {
    std::cout << "ns1::bar()\n";
}

namespace ns2 {
void bar() {
    std::cout << "ns1::ns2::bar\n";
}
}  // namespace ns2

namespace ns3 {
void botva_ns3() {
    std::cout << "botva_ns3()\n";
}
}  // namespace ns3

namespace ns3::ns4 { // просто сделали короче, чтобы не писать ns3 внутри котрого писать ns4
namespace ns1 {  // ns1::ns3::ns4::ns1 
    // !!!ДА, просто такое же имя, но другой namespace
}

void baz() {
    botva_ns3();  // unqualified name lookup for 'botva_ns3'
    ns2::bar();  // unqualified name lookup for 'ns2', qualified name lookup bar()
    // ns2::foo();  // compilation error: no 'foo' inside 'ns2'
    // ns1::ns2::bar();  // thinks that 'ns1' is 'ns1::ns3::ns4::ns1', 'n2' not found
    ::ns1::ns2::bar(); // qualified 
}
}  // namespace ns3::ns4
}  // namespace ns1

int main() {
ns1::ns3::ns4::baz();
}
```

Кста qualified - это найти при помощи unqualified нужное место, а потом просто посмотреть внутрь.

Unqualified - он поднимается по уровнням наверх, когда найдет - остановится.(а лол это 13 билет, но тут без этого - никуда)

Мем: если namespce на одном уровне вложенности и называются одинаково, то это один namespace(ns3), если на разных, но название совпало, то это разные(s1 и s1::s3::s4::s1) 

Читаем комменты, кто проникся? 

По факту надо что запомнить: если без двоеточия в начале, то мы поднимаемся наверх, пока не найдем че хотели(интересует первый раз когда найдем первый наймспейс в пути(самую левую в объявлении)) потом - тупо идем вниз по остатку пути

Если есть двоеточие, то идем на самый верх(глобальную мусорку) и спускаемся с нее по пути
* ### Псевдонимы типов typedef, using

Кароче просто хотим по-своему типы называть, есть 2 способа(дефайн для лохов), который чисто синтаксически различаются(
typedef неудобный, но обратная совместимость). После компиляции - не отличить псведоним от исходного

Егор на вопрос в чем разница: "Но вроде есть какой-то баг в
GCC: https://stackoverflow.com/questions/48613758/using-vs-typedef-is-there-a-subtle-lesser-known-difference"

```c++
#include <vector>
#include <utility>
#include <typeinfo>
#include <iostream>

typedef std::vector<int> vi;
using pii = std::pair<int, int>;
// Please do not #define: it does not respect namespaces/private/public
// https://stackoverflow.com/a/1666375/767632

int main() {
    vi v1(10);
    std::vector<int> &v2 = v1;
    std::cout << typeid(v1).name() << "\n";
    std::cout << typeid(v2).name() << "\n";

    pii p1(10, 20);
    std::pair<int, int> &p2 = p1;
}
```

* ### Че по using namespace

Обычно пишется внутри какого-то отдельного кусочка программы (то есть не глобально), потому что иначе можем произойти
коллизия имен. Прикольны пример [04-210923/01-functions/02-lcm](https://github.com/hse-spb-2021-cpp/lectures/blob/master/04-210923/01-functions/02-lcm.cpp). Там мы делаем лонговый лцм, но вызываем от интов,
вызовется стандтартная функция(спс namespace std), мы там переполняемся и дохнем.
