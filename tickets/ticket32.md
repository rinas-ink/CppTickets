# Билет 32 (One Definition Rule и нарушения IFNDR)

## ODR

ODR (One Definition Rule) - во всей программе (то есть включая все файлы) у любой сущности должно быть ровно одно определение. 
Определение с cppreference: Only one definition of any variable, function, class type, enumeration type, concept (since C++20) or template is allowed in any one translation unit (some of these may have multiple declarations, but only one definition is allowed).

### Что с перегрузкой функций
#### Напоминание про перегрузку функций: 
Есть несколько функций с одинаковым именем, но с разными типами аргументов. Тогда компилятор может выбрать наиболее подходящую перегрузку.
#### Что происходит внутри:
API (Application Programming Interface) - показывает, какой программный интерфейс у различных translation unit'ов. (Какие типы у аргументов функции, какие типы возвращаются, в каком namespace лежит и тд). Всё API запоминает компилятор и им можно пользоваться. (Подробнее - https://youtu.be/X-6unqJz_uY?list=PL8a-dtqmQc8obAqSKqGkau8qiafPRCxV7&t=1683)
```C++
void foo();
foo();
```
При компиляции все превращается в ABI (Application Binary Interface) - тоже самое, но более низкоуровневое. У компилятора есть регламенты, как и через что возвращается (через какие регистры процессора, через какие места памяти и тд). Например, поэтому нельзя компилировать разные части программы разными компиляторами, в итоге какие-то регламенты могут не совпасть.  
Затем происходит name mangling:
```C++
void foo(); -->  \_Z3foov // v - тип аргумента. 
```
Все это к тому, что перегруженные функции различаются компилятором и нарушения ODR не возникает. Пример:  
foo.cpp:
```C++
void foo(int) {
}
```
main.cpp:
void foo(int);
void foo() {
}

int main() {
    foo();
    foo(10);
}

### Пример ошибок линковщика
#### Multiple definition 
Функция имеет более одного определения.  
foo.cpp:
```C++
void foo() {
}
```
main.cpp:
```C++
void foo() {
}

int main() {
}
```
#### Undefined reference
Функция не имеет опрделения, но при этом где-то используется.
```C++
int main() {
    foo();
}
```
### Примеры IFNDR
IFNDR (Ill-Formed, No Diagnostic Required) - "
#### Несовпадение объявлений функций

С аргументами по умолчанию:  
foo.cpp:
```C++
void foo(int x = 10) {
    std::cout << x << "\n";
}
```
main.cpp:
```C++
void foo(int = 1000);

int main() {
    foo();
}
```
Возвращаемое значение:  
foo.cpp:
```C++
float foo() {
    return 1000000;
}
```
main.cpp:
```C++
#include <iostream>

double foo();

int main() {
    std::cout << foo();
}
```






















