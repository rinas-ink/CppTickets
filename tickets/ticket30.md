## Билет 30. Препроцессор

**`Note`**:

У `g++` можно запустить только препроцессор, для этого используйте ключ `-E`: `g++ -E main.cpp >main`.

Здесь перечислены имена переменных, которые нельзя использовать: [StackOverFlow answer](https://stackoverflow.com/questions/228783/what-are-the-rules-about-using-an-underscore-in-a-c-identifier/228797#228797)



### 1. Как пользоваться `#ifdef DEBUG`, `#define` и ключом `-DDEBUG` (в случае `GCC/Clang`):

1. `#ifdef ANY_VAR` - здесь мы проверяем, если переменная препроцессора `ANY_VAR` объявлена, то сделать то, что идет за `#ifdef`:

	Здесь если `DEBUG` **объявлена**, то часть с `#else` будет вырезана из файла и компилятор ее даже не увидит (если `DEBUG` **не объявлена**, то наоборот - останется часть в `#else`):

	```cpp
	#ifdef DEBUG
		std::cout << "Debugging mode is on!";
	#else // optional else statement
		std::cout << "Debugginh mode is off!";
	#endif
	```

2. `#define SOME_VAR value` - здесь мы объявляем (объявляем именно для препроцессора, а не для языка `C++`) переменную `SOME_VAR` и говорим препроцессору заменить все вхождения слова `SOME_VAR` в коде на `value` - вырезать все `SOME_VAR` и вставить конкретно слово `value` (вместо `value` могло стоять все, что угодно).

3. `-DDEBUG` - данный ключ пишется в момент компиляции программы, с помощью него мы объявляем переменную `DEBUG` (то есть это эквивалентно тому, чтобы написать `#define DEBUG 1` в коде).




### 2. Базовый синтаксис для `#if` и `defined()`:

1. `#if` - используется для более сложных условий проверки.

2. `defined(ANY_PREPROCESSOR_VAR)` - проверяет, объявлена ли препроцессорная переменная с именем `ANY_PREPROCESSOR_VAR`.

3. Пример использования:

	```cpp
	#if defined(SOME_VAR) && 10 * 3 <= 50
		std::cout << "SOME_VAR defined and 30 is less than 50!";
	#endif	
	```

4. Есть еще `#elif` - нужен, чтобы писать много разных условий:

	```cpp
	#if defined(FIRST)
		std::cout << "first defined";
	#elif defined(SECOND)
		std::cout << "second defined";
	#else
		std::cout << "maybe third?..."
	```




### 3. Определение компилятора и компилятороспецифичные `#pragma` (например, игнорирование предупреждений в `GCC`):

1. Переменная `__cpluplus` - содержит в себе информацию о версии языка (возможные значения `201700L`, `201400L`), с которой скомпилирован файл.

2. Определяем версию компилятора:

```cpp
#include <iostream>

int main() {
	std::cout << "C++ " << __cplusplus << "\n";

	std::cout << "Detecting compiler...\n";
#if defined(__GNUC__) && !defined(__clang__)
	std::cout << "GNU C " << __GNUC__ << "." << __GNUC_MINOR__ << "." << __GNUC_PATCHLEVEL__ << "\n";
#endif
#ifdef __clang__
	std::cout << "LLVM clang " << __clang_version__ << "\n";
#endif
#ifdef _MSC_VER
	std::cout << "Microsoft Visual C++ " << _MSC_VER << "\n";
#endif

	std::cout << "Detecting standard library...\n";
#ifdef __GLIBCXX__
	std::cout << "GNU libstdc++: " << __GLIBCXX__ << "\n";
#endif
#ifdef _LIBCPP_VERSION
	std::cout << "LLVM libc++: " << _LIBCPP_VERSION << "\n";
#endif
#ifdef _MSVC_STL_VERSION
	std::cout << "Microsoft STL: " << _MSVC_STL_VERSION << " at " << _MSVC_STL_UPDATE << "\n";
#endif
}
```

3. `#pragma` - команда компилятору о том, что надо что-то сделать (не имеет отношения к препроцессору). Это все специфично для каждого компилятора, поэтому компиляторы не понимают `#pragma` друг друга.

Игнорирование предупреждений в `GCC`:
```cpp
#pragma GCC diagnostic ignored "-Wunused-variable" // команда для GCC о том, чтобы он игнорировал неиспользуемые переменные
```

```cpp
int main() {
	#ifdef __GNUC__  // Everything is compiler-specific
		#pragma GCC diagnostic push // кладем все предупреждения на специальный стек
		#pragma GCC diagnostic ignored "-Wunused-variable" // говорим игнорировать предупреждения 'unused-variable'
	#endif
		int x = 10;
	#ifdef __GNUC__
		#pragma GCC diagnostic pop // восстанавливаем старые предупреждения
	#endif
}
```




### 4. Проблемы с аргументами макросов: приоритет операторов, повторное вычисление.

1. **Приоритет операторов**: так как препроцессор **только заменяет** все вхождения объявленной переменной на указанное значение, то важно самому следить за приоритетом операций:

	```cpp
	#define A 2 + 2
	std::cout << A * 4; // equals to 2 + 2 * 4 = 10, not (2 + 2) * 4 = 16 
	```

	```cpp
	#define max(a, b) a < b ? a : b
	std::cout << max(1, 2); // compilation error, because treating like '(cout << 1) < (2 ? 1 : 2)' - comparing 'iostream' and 'int' 
	```

	```cpp
	#define mul(a, b) (a * b)
	std::cout << mul(2 + 1, 3); // prints 2 + 1 * 3 = 5, not (2 + 1) * 3 = 9
	```

2. **Повторное вычисление**:
	
	```cpp
	#define max(a, b) ((a) < (b) ? (a) : (b))
	
	int x = 10;
	
	max(x++, 10) // UB: incrementing x twice in one statement (because (a) -> (x++) and we do it twice)

	max(foo(), 10) // calling foo() twice: (a) -> (foo())
	```



### 5. Проблемы с макросами из нескольких `statement` и трюк с `do { } while(0)`

Примеры проблем:

```cpp
// #1
#define print_twice(x) cout << x; cout << x;

// problems with:
int x = 10;
if(x == 10)
	print_twice(x); // only one print
```

```cpp
// #2
#define print_twice(x) { cout << x; cout << x; }

// problems with:
int x = 10;
if(x == 10)
	print_twice(x); // only one print
else
	print_twice(0);

// compilation error, expands to:
/*
if(x == 10)
	{ cout << x; cout << x; }; <- this ';' causes error
else
	{ cout << 0; cout << 0; };
*/
```

Трюк с `do {} while(0)`: мы вынуждены писать так, чтоюбы примеры выше работали:

```cpp
#define print_twice(x) do { cout << x; cout << x; } while(0) // no ';' at the end!

// BUT! following code causes error, because 'do{} while()' is not expression, it is statement (no solution for the problem):

print_twice(10), print_twice(10); // ERROR: while is not expression
```





### 6. Проблемы с `operator`, и вызовом макросов: как добиться, чтобы `assert(true)`, `assert(false)` работало:

Проблема в том, что если сделать `assert` через `if` или любое другое выражение (`statement`), то мы не сможем писать код по типу:
```cpp
assert(12 == 6*2), assert(1 == 1); // ERROR! statements cannot be used with operator,()
```

Код выдает ошибку, так как `statements` нельзя использовать с операторами. Чтобы избавиться от ошибки, необходимо писать `assert` через какой-нибудь `expression`: **функция**, **тернарный оператор**:

```cpp
// #1 with function implementation
bool evaluate(bool expr) {
	return expr;
}
#define assert(expr) evaluate((expr))


// #2 with ternary operator
#define assert(expr) ((exprt) ? true : false)
```





### 7. Вариадический макрос: `__VA_ARGS__` и `...`, проблемы с последней запятой:

1. На примере:

```cpp
#define foo(a, b, ...) bar(b, a, __VA_ARGS__)

int main() {
	foo(1, 2, 3, 4); // expands to: bar(2, 1, 3, 4)
	foo(1, 2, 3); // expands to: bar(2, 1, 3)
}
```

2. Проблема с последней запятой:

```cpp
#define foo(a, b, ...) bar(b, a, __VA_ARGS__)

int main() {
	foo(1, 2); // expands to: bar(2, 1, ) -> ERROR
}
```

3. Решение проблемы с запятой появилось только в `C++20`:
```cpp
// solution appeared only in C++20: __VA_OPT__()
// rewrite foo:
#define foo(a, b, ...) bar(b, a __VA_OPT__(,) __VA_ARGS__)
```






### 8. Проблемы с `assert(v == std::vector{1, 2, 3})`, `foo({1, 2})`, `foo(map<int, int>{})` и как их решать:

**Как их решать**: решение у всех почти одно, сначала используем `__VA_OPT__(,)`, а далее при вызове `foo()` оборачиваем аргументы в круглы скобки вот так `foo( (map<int, int>) )`, тогда разрезаний по `,` в типе `map<int, int>` не будет.

1. `assert(v == std::vector{1, 2, 3})`:

	**Проблема:**

	```cpp
	vector<int> v = {1, 2, 3};
	assert(v == vector{1, 2, 3}); // compilation error, cannot parse vector correctly
	```

	**Решение:**

	```cpp
	vector<int> v = {1, 2, 3};
	assert(v == (vector{1, 2, 3}) ); // works fine, due to embracing vector in brackets
	```

2. `foo({1, 2})`:

	**Проблема:**
	```cpp
	#define foo(a, b, ...) bar(b, a, __VA_ARGS__)

	int main() {
		foo({1, 2}); // expands to: bar(2}, {1, )

		foo({1, 2, 3, 4}); // expands to: bar(2, ({1, 3, 4})) - treating '{' as part of parameter			
	}
	```

	**Решение:**
	```cpp
	// using __VA_OPT__()
	#define foo(a, b, ...) bar(b, a __VA_OPT__(,) __VA_ARGS__)

	int main() {
		foo( ({ 1, 2 }), 1 ); // expands to: bar(1, ({1, 2}) )
	}
	```

3. `foo(map<int, int>{})`:

	**Проблема:**
	```cpp
	#define foo(a, b, ...) bar(b, a, __VA_ARGS__)

	int main() {
		foo(map<int, int>{}); // expands to: bar(int>{}, map<int, )
	}
	```

	**Решение:**
	```cpp
	#define foo(a, b, ...) bar(b, a __VA__OPT__(,) __VA_ARGS__)		

	int main() {
		foo(1, (map<int, int>{}) ); // expands to: bar( (map<int, int>{}), 1 )
	}
	```






### 9. Как использовать `#macro_arg`, `__FILE__`, `__LINE__` для создания своего макроса `CHECK` в тестовом фреймворке:

1. `#macro_arg` - здесь `#` преобразует переданный параметр макроса `marco_arg` в строку, то есть сделает нам `"marco_arg"`:

```cpp
#define print(x) (cout << #x)

print(output some text); // expands to: (cout << "output some text"); 
``` 

2. `__FILE__` - название текущего файла.

3. `__LINE__` - текущая строка в файле.

4. Как использовать в своем фреймворке для `CHECK`:
```cpp
// printOnError function:

void printOnError(bool hasError, string expr, string filename, int line) {
	if(hasError) {
		cout << expr << "has failed in file: " << filename << ", on line: " << line << endl;
	}
}
```
```cpp
#define CHECK(expr) \
	printOnError((!(expr)), #expr, __FILE__, __LINE__)
```







### 10. Базовое использование `##` в макросах:

`##` - склеивает 2 переменные/параметра вместе, например:
```cpp
#define glue(a, b) a ## b // whitespaces between 'a' and 'b' omitted

int xy = 12;
cout << glue(x, y); // expands to: cout << xy; which prints '12' in console
```

1. Было: генерация случайных имён переменных/функций при помощи `__LINE__` (`exercises/05-211004/10-new-var`):

```cpp
#define CONCAT_IMPL(x, y) x##_##y
#define CONCAT(x, y) CONCAT_IMPL(x, y)

// suppose executing on 10th line:
void CONCAT(func, __LINE__)(); // expands to: 'void func_10();'
```







### 11. Возможность писать в макросах лишь куски корректного кода:

1. Если мы пишем `#if ... #elif ... #else`, то лишь 1 часть из этих `if`-ов выполнится, а остальные будут вырезаны из программы препроцессором до этапа компиляции, поэтому если в каком-нибудь из `if`-ов был написан некорректный код на языке `C++`, но данный `if` не выполнился, то ошибки при компиляции не будет, так как данной части кода просто не будет в файле после работы препроцессора.

2. В лабе `TEST_CASE` выводит нам примерно следующий код `void test()`, а далее ожидается, что вызвавший макрос `TEST_CASE` напишет фигурные скобки и определит в них тело функции-теста, то есть получается, что сам макрос выдает некорректный код: мы обязаны дописывать тело функции, чтобы все компилировалось.



#### Общая идея создания фреймворка для автотестов с авторегистрацией тестов: `TEST_CASE("hi") { CHECK(1 == 2); }`:

1. Мы создаем функцию со статической переменной-вектором, которая будет хранить в себе ссылки на функции (функции - это сами тесты). Эта функция будет по ссылке возвращать нам данный массив, чтобы мы могли класть в него ссылки на тесты (мы делаем это как функцию, так как порядок инициализации переменных между единицами трансляции не задан, поэтому просто статический массив мог не успеть определиться до создания первого теста).

2. `TEST_CASE` делается следующим образом:

```cpp
#define TEST_CASE(name) \
static void CONCAT(func, __LINE__)(); \
const FunctionSaver CONCAT(saver, __LINE__)(name, CONCAT(mytest_func, __LINE__)); \
void CONCAT(mytest_func, __LINE__)()

// suppose calling on line 10
// expands to: 

static void func_10();
const FunctionSaver saver_10(name, func_10);
void func_10()


// use of TEST_CASE
TEST_CASE("test") { // on line 10
	...
}

// expands to 
static void func_10();
const FunctionSaver saver_10(name, func_10);
void func_10() {
	...
}
```

3. Здесь `saver_10` получает в конструктор ссылку на функцию `func_10()`, далее сохраняет ее в глобальный статический массив тестов. Важно, чтобы `saver_10` и `func_10()` были видны только в текущей единице трансляции, это достигается за счет `const` для переменных (здесь это `saver_10`) и `static` для функций (здесь `func_10()`).

4. После того, как все тесты собраны в массив, мы в каком-нибудь `main()` проходимся по всем этим тестам и вызываем каждый из них.