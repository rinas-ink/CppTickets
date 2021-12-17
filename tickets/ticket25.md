## Функторы и лямбды  

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
	std::cout << comparator(-8);                       // true
	std::cout << comparator(100, 99);                  // true

	std::cout << Greater().operator()();               // true
	std::cout << Greater()(-8);                        // true
	
	std::vector<int> vec{1, 2, 3, 4, 5, 6, 7, 8}; 
	std::sort(vec.begin(), vec.end(), comparator);          // will reverse vector
}

```

`operator()` логично помечать `const`, т.к. обычно функторам не нужно мутировать состояние


### использование функторов

В примере выше функтор передаётся в `std::sort` в качестве компаратора. Так же его моджно передать в `std::set`. Т.к.
set хочет скопировать компаратор в себя (как и почти всё в STL) и считает, 
что компаратор не может менять состояние при вызове, так что`operator()` придётся пометить `const`. 

```c++
int levenshtein_ditance(std::string const& a, std::string const& b);


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

Компараторы можно складывать в  `std::function`, при этом он копируется на стэк (ну если там нет small object optimization)

Для хранения компараторов есть тип `std::reference_wrapper`, который позволяет сохранять ссылки на функторы в `std::vector` или
где-нибудь ещё и прокидывает наверх `operator()`.

Канонический способ создания `std::reference_wrapper` -  `auto func = std::ref(functor);`

У этого типа есть метод `get`, возвращающий обычную ссылку на объект, на который ссылается `std::reference_wrapper`.


**жирный текст** _курсив_ ~~ошибка~~ ==мнения?== `inline code block` [ссылка на фулл](https://www.markdownguide.org/cheat-sheet/)
```c++
int main() {
    return 0;
}
```

### Подзаголовок

LMAO
Not bottom text, lol
