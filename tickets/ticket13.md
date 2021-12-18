## Билет 13. Подробности именования сущностей
- ###[[maybe_unused]] 
используется для уведомления компилятора о том, что
сущность может быть не использована в программе и следует подавлять
соответствующее предупреждение.
- ###The most vexing parse
    Текста внутри оч мало, не боимся.
    - [сэнкс что уже есть](https://github.com/vladnosiv/hse-spb-conspects-2020/blob/master/C%2B%2B/ticket32.md#the-most-vexing-parse)
    - [примеры с наших лекций, нужен 2 файл](https://github.com/hse-spb-2021-cpp/lectures/tree/master/05-210930/02-declare-define)
    - [про структурки, с лекции](https://github.com/hse-spb-2021-cpp/lectures/tree/master/06-211006/00-past)
- ###Допустимые имена переменных, функций, констант, классов:
    - Нельзя начинать с цифры.
    - Ключевые слова - бан([список](https://en.cppreference.com/w/cpp/keyword))
    - C `_` кеки(UB)([все ошибки](https://stackoverflow.com/questions/228783/what-are-the-rules-about-using-an-underscore-in-a-c-identifier)):
        * Нельзя с него начинать глобальные имена
        * Двойное => бан(только компилятор так может)
        * Начинаем с `_`, а потом заглавная буква.
        * `_t` - не UB(по стандарту), но POSIX(бубнта и макосось) по кеку зарезервировал некоторые имена, можно огрести
    - Последствия - повезло/UB самые разные кеки[(пример с кфа про _end)](https://codeforces.com/blog/entry/17747)
- ###Structured binding для пар, простых структур, массивов, со ссылкой
    [ссыль на пример лекции](https://github.com/hse-spb-2021-cpp/lectures/blob/master/03-210916/01-extra-stl/02-structured-binding.cpp)
    ```c++
    auto [a, b, c] = std::tuple(32, "hello"s, 13.9);
    auto [a, b] = foo();//foo() возвращает пару
    ///пара со ссылкой
    std::pair<int,int> f;
    auto &[a, b] = f; //тогда a/b привяжутся к её first/second
    /////простые структуры
    struct Point {int x, y;};
    Point p = { 1,2 };
    auto[ x, y] = p;
    //фиксированные массивы
    int arr[3] = { 1, 2, 3 };
    auto[x, y, z] = arr;
    ```
    - minmax trouble
    
    Функция возвращает пару, но паруу ссылок на значения, а биндинг просто копирует, ну молодец, огребай
    ```c++
    {
        std::pair<int, int> p = std::minmax(30, 20);
        std::cout << p.first << " " << p.second << "\n";  // Ok!
    }
    {
        auto [x, y] = std::minmax(30, 20);
        std::cout << x << " " << y << "\n";  // UB
    }
    ```
- ###Поиск имён
    - Квалифицированный и неквалифицированный поиск, порядок обхода вложенных namespace
        
        [вай +10 минут сна мне](https://github.com/vladnosiv/hse-spb-conspects-2020/blob/master/C%2B%2B/all-tickets.md#%D0%B1%D0%B8%D0%BB%D0%B5%D1%82-03-%D0%BF%D1%80%D0%B0%D0%B2%D0%B8%D0%BB%D0%B0-%D0%BF%D0%BE%D0%B8%D1%81%D0%BA%D0%B0-%D0%B8%D0%BC%D1%91%D0%BD-%D0%B3%D0%BB%D0%BE%D0%B1%D0%B0%D0%BB%D1%8C%D0%BD%D1%8B%D1%85-%D0%B8-%D0%B2%D0%BD%D1%83%D1%82%D1%80%D0%B8-%D0%BA%D0%BB%D0%B0%D1%81%D1%81%D0%BE%D0%B2)
    - Отличие между std:: и ::std::
      Если вы пишете нормальный код и его не будут исопльзовать через левое ухо, то это одно и тоже. 
      
      НО если кто-то сделает стурктуру/namespace с именем std(он обязательно в каком-то namespace, тк в std нельзя ничего вставлять, это UB)
      То может произойти примерно вот такой кек: 
      ```c++
        #include <iostream>
        int main() {
          struct std{};
          std::cout << "fail\n"; // Error: unqualified lookup for 'std' finds the struct
          ::std::cout << "ok\n"; // OK: ::std finds the namespace std
        }

      ```
    - ADL (argument-dependent lookup) для операторов и функций
      Видимо мы еще не прошли таааак глубоко [Версия от Влада](https://github.com/vladnosiv/hse-spb-conspects-2020/blob/master/C%2B%2B/all-tickets.md#%D0%B1%D0%B8%D0%BB%D0%B5%D1%82-04-adl) - Там чет сильно больше, чем у нас было(вроде), 
      
    попробую нашу версию:
      
      ```c++
      // Argument-Dependent Lookup aka Koenig Lookup

        namespace ns {
        struct Foo {};
        
        void do_something() {}
        void do_something(Foo) {}
        bool operator==(const Foo&, const Foo&) { return true; } // то ради чего все
        };
        
      int main() {
        // do_something(); - а шо ты хочешь когда так пишешь?
        ns::do_something(); // ну так можно
    
        // Foo f;
        ns::Foo f; // Так заработает тк теперь мы будем смотреть на все наймспейсы где лежат аргументы
        do_something(f);  // unqualified name lookup, ADL enabled
       
        // вот он, настоящий пельмень:
        // f == f;
        // operator==(f, f)
        // ns::operator==(f, f);
    
        // Example:
        // getline(std::cin, str)
        // Better: std::getline
        //
        // std::vector<int> v{1, 2, 3};
        // sort(v.begin(), v.end());  // v.begin() ~ std::vector<int>::iterator ~(?) int*
        // Better: std::sort
      }
    
      ```
    - Hidden friend
      
      friend-function's внутри пространства имен, которые объявлены в связанном классе видны для ADL, даже если не видны для других поисков.
      
      [Влад гений](https://github.com/vladnosiv/hse-spb-conspects-2020/blob/master/C%2B%2B/all-tickets.md#hidden-friend-%D0%BF%D1%80%D0%B0%D0%BA%D1%82%D0%B8%D1%87%D0%B5%D1%81%D0%BA%D0%BE%D0%B5-%D0%BF%D0%BE%D1%81%D0%BE%D0%B1%D0%B8%D0%B5-%D0%BF%D0%BE-%D0%BF%D0%BE%D0%B8%D1%81%D0%BA%D1%83-%D0%B4%D1%80%D1%83%D0%B7%D0%B5%D0%B9)
      
      [Еще в 23 билете про это классно написали, ну и просто инфа про друзей итп есть](https://github.com/khbminus/CppTickets/blob/master/tickets/ticket23.md#%D0%B4%D1%80%D1%83%D0%B7%D1%8C%D1%8F-%D1%84%D1%83%D0%BD%D0%BA%D1%86%D0%B8%D0%B8)
- ###Shadowing переменных в рамках одной функции

(скоммуниздил с [хабра](https://habr.com/ru/company/vk/blog/341584/), use ctrl+f)
  
Скрытие переменной (Variable shadowing) происходит, когда переменная, объявленная в одной области видимости (например, блоке или функции), имеет такое же имя, как и другая переменная, определённая во внешней области видимости. Тогда внешняя переменная будет скрыта внутренней
```c++
bool x = true;                                              // x is a bool
auto f(float x = 5.f) {                                     // x is a float
    for (int x = 0; x < 1; ++x) {                           // x is an int
        [x = std::string{"Boo!"}](){                        // x is a std::string
            { auto [x,_] = std::make_pair(42ul, nullptr);}  // x is now unsigned long
        }();
    }
}
```  

        