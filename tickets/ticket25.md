## Билет 25. Функторы и лямбды  

Функциональный объект, иногда называемый функтор - что угодно, к чему применима семантика вызова, как у функции.

Функторами могут быть:

* функции
* указатели на функции 
* объекты с перегруженным `operator()`

### `operator()`

```c++

struct Greater {
	bool operator()(int a, int b) const {
		return a > b;
	}
	bool operator()(int a) const {                    // overloads are possible
	    return a > 0;
	}
	bool operator()() const {
	    return true;
	}
};

int main() {
	const Greater comparator;

	std::cout << comparator.operator()();              // true
	std::cout << comparator.operator()(-8);            // false
	std::cout << comparator.operator()(100, 99);       // true

	std::cout << comparator();                         // true
	std::cout << comparator(-8);                       // false
	std::cout << comparator(100, 99);                  // true

	std::cout << Greater().operator()();               // true
	std::cout << Greater()(-8);                        // false
	
	std::vector<int> vec{1, 2, 3, 4, 5, 6, 7, 8}; 
	std::sort(vec.begin(), vec.end(), comparator);     // will reverse vector
}

```

`operator()` логично помечать `const`, т.к. обычно функторам не нужно мутировать состояние


### использование функторов

В примере выше функтор передаётся в `std::sort` в качестве компаратора. Так же его моджно передать в `std::set`. Т.к.
set хочет скопировать компаратор в себя (как и почти всё в STL) и считает, 
что компаратор не может менять состояние при вызове, так что`operator()` придётся пометить `const`. 

```c++
int levenshtein_distance(std::string const& a, std::string const& b);


struct Closer {
    std::string template;

    bool operator()(std::string const& a, std::string const& b) const {  // 'const' is important
        return levenshtein_ditance(a, template) < levenshtein_ditance(b, template);
    }
};

int main() {
    using namespace std::literals;
    std::set<std::string, Closer> v1({"meow"s, "woof"s, "oink"s, "quack"s}, Closer{"meow"s});
    std::set<std::string, Closer> v2({"meow"s, "woof"s, "oink"s, "quack"s}, Closer{"woof"s});
}

```

При присвоении `std::set` компараторы тоже поменются. 

Обычно аргументы в `operator()` передают по констанной ссылке - очень странно, если компаратор хочет изменить свои аргументы, 
а копирование занимает время.  

Заметим, что из-за того, что большинство алгоритмов STL принимает функторы с семантикой копирования, при передаче 
функтора, как аргумента в алгоритм `std::all_of`, `std::any_of`, `std::for_each` и т.д. состояние функтора никак не поменяется.
`std::for_each` возвращает функтор после применения его к аргументам, так что в его состоянии всё-таки можно что-то хранить.

```c++
struct Functor {
	int val = 0;
	void operator()(int) {
		val++;
	}
};

int main() {
	Functor c;
	std::vector<int> v(100, 0);
	auto used = std::for_each(v.begin(), v.end(), c); // Если operator() менял состояние c, то этот вызов отразится на c
	std::cout << c.val << std::endl;                  // 0
	std::cout << used.val << std::endl;               // 100
}

```

Функторы можно складывать в  `std::function`, при этом он копируется в объект с dynamic storage duration
(ну если там нет small object optimization)

Для хранения ссылок на функторы есть тип `std::reference_wrapper` из библиотеки `functional`, 
который позволяет сохранять ссылки на функторы в `std::vector` или
где-нибудь ещё и прокидывает наверх `operator()`, оставляя возможность вызывать функтор, лежащий по ссылке.
При этои если у хранимого объекта нет такого оператора с подходящей сигнатурой код не скомпилируется (см. perfect forwarding
и ленивое раскрытие шаблонов)

Канонический способ создания `std::reference_wrapper` -  `auto func = std::ref(functor);`

У этого типа есть метод `get`, возвращающий обычную ссылку на объект, на который ссылается `std::reference_wrapper`.

Теперь функторы можно передавать в алгоритмы stl


```c++
struct Functor {
	int val = 0;
	void operator()(int) {
		val++;
	}
};

int main() {
	Functor c;
	std::vector<int> v(100, 0);
	std::for_each(v.begin(), v.end(), std::ref(c)); // Если operator() менял состояние c, то этот вызов отразится на c
	std::cout << c.val << std::endl;                // 100
}
```
`std::reference_wrapper` не владеет, тем на что ссылается и может быть висячим.

### лямбды

Лямбда-выражения - синтаксический сахар для создания функторов. Тип созданного функтора невозможно описать,
поэтому сохранять объекты этого типа можно только через `auto` или, если
выражение такого типа уже можно написать, то при помощи `decltype`. Синтаксис лямбда-выражения `[<capture>](<arguments>){<body>}`.
Тип возвращаемого функтором значения выводится сам (на самом деле его можно задать используя синтаксис `-> Type`).


Поля получившегося объекта функтора описываются в замыкании:

* `[=]` - все используемые в теле выражения переменнные из текущего scope копируются внутрь функтора.
* `[&]` - все используемые в теле выражения переменнные хранятся в функторе по ссылке.
* `[this]` - текущий `this` захватывается по ссылке
* `[\*this]` - текущий `this` захватывается по значению
* `[name = expression]` - создаём у функтора новое поле name, присваиваем туда `expression` (тип поля выведется сам)
* `[&name = expression]` - создаём у функтора новое поле=ссылку name на `expression` (тип поля выведется сам)
* `[name]` - копируем в функтор переменную `name`
* `[&name]` - храним в функторе ссылку на переменную `name`

Эти замыкания можно комбинировать всеми способами, более конкретные перекрывают менее конкретные.

Про `[&name = expression]` нужно отметить, что никак пометить в замыкании, что ссылку нужно брать именно константную не выйдет,
поэтому нужно сделатьт так, стобы тип `expression` сожержал категорию константности. Стандартый способ сделать это
`[&name = std::as_const(expression)]` ну или `[&name = static_cast<Type const&>(expression)]`.

По умолчанию `operator()` у получившегося функтора помечен, как `const`. Избежать этого можно, добавив слово `mutable` перед телом лямбды.

Примеры

```C++
int main() {
    int a = 0, b = 1, c = 2;
    auto first = [=, &a]() mutable {
	     std::cout << a << " " << b++ << " " << c << std::endl;
	};
    decltype(first) second = first;
    first();                             // 0 1 2            
    first();                             // 0 2 2
    second();                            // 0 1 2
    a = b = c = 100;
    first();                             // 100 3 2
    second();                            // 100 2 2
}
```


```C++
int main() {
    auto lambda = [val = 0](int) mutable { std::cout << val++ << '\n'; };
    std::vector<int> v(100, 0);
    std::for_each(v.begin(), v.end(), lambda); // numbers from 0 to 99
}
```

[раскрывается так](https://cppinsights.io/lnk?code=I2luY2x1ZGUgPGlvc3RyZWFtPgojaW5jbHVkZSA8dmVjdG9yPgojaW5jbHVkZSA8ZnVuY3Rpb25hbD4KCmludCBtYWluKCkgewogICAgYXV0byBsYW1iZGEgPSBbdmFsID0gMF0oaW50KSBtdXRhYmxlIHsgc3RkOjpjb3V0IDw8IHZhbCsrIDw8ICdcbic7IH07CiAgICBzdGQ6OnZlY3RvcjxpbnQ+IHYoMTAwLCAwKTsKICAgIHN0ZDo6Zm9yX2VhY2godi5iZWdpbigpLCB2LmVuZCgpLCBsYW1iZGEpOyAvLyBudW1iZXJzIGZyb20gMCB0byA5OQp9Cg==&insightsOptions=cpp17&std=cpp17&rev=1.0)

Поля структуры нельзя захватить по отдельно от всего объекта - приходится захватывать `[this]` или `[*this]`.

```c++

struct Foo {
  int a;
  auto copy_capture() {
  	return [*this](){
      std::cout << a << std::endl;
    };
  }
};

int main()
{
  auto lambda1 = Foo{1}.copy_capture();
  lambda1(); // no dangling reference
}

```

[раскрывается так](https://cppinsights.io/lnk?code=I2luY2x1ZGUgPGlvc3RyZWFtPgoKc3RydWN0IEZvbyB7CiAgaW50IGE7CiAgYXV0byBjb3B5X2NhcHR1cmUoKSB7CiAgCXJldHVybiBbKnRoaXNdKCl7CiAgICAgIHN0ZDo6Y291dCA8PCBhIDw8IHN0ZDo6ZW5kbDsKICAgIH07CiAgfQp9OwoKaW50IG1haW4oKQp7CiAgYXV0byBsYW1iZGExID0gRm9vezF9LmNvcHlfY2FwdHVyZSgpOwogIGxhbWJkYTEoKTsgLy8gbm8gZGFuZ2xpbmcgcmVmZXJlbmNlCn0=&insightsOptions=cpp17&std=cpp17&rev=1.0)


Довольго популярна идиома передачи в лямбды `[field = std::move(value)]` - 	это, например, единственный способ передать
в лямбду `std::unique_ptr`. Растаманского `move` тут нет, поэтому каждое такое поле придётся перемещать руками.
