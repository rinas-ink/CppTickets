# 18. `unique_ptr`, управление памятью, move
* ## Ручное управление памятью: `new`, `delete`, `new[]`, `delete[]`, когда что использовать.
Это `Dinamic storage duration`, `new` выделяет память, обычно на куче, и возвращает указатель на свежесозданный объект, который мы обязуемся удалить с помощью оператора `delete`. (иначе произойдёт утечка памяти). Квадратные скобки выделяют массив элементов, которые будут лежать подряд в памяти. В скобках нужно указать количество объектов в массиве. При этом при удалении этого делать не нужно.  
```C++
struct Foo{

};
int main() {
    Foo *f = new Foo;
    Foo *arr = new Foo[1000];
    delete f;
    delete[] arr; // no memory leak
}
```
  * ### Не было: разница между `new int;` и `new int();`
  * ### Утечка памяти: UB ли, какие последствия, как ловить, как читать вывод sanitizer и Valgrind с примерами.  
  Утечка памяти это не уб, программа спокойна работает, просто появляется всё новая и новая выделенная память, что может вызвать проблемы. Например, если мы сервер вк, то утечки памяти лучше не допускать, иначи со временем памят на сервере кончится. Ловить с помощью `valgrind` или санитайзеров.  
  ```C++
    struct Foo{
        int a;
    };
    int main() {
        Foo *f = new Foo; // Memory leak
        return 0;
    }
  ```
  Запускаем валгринд командой `valgrind ./a`. Он напишет сколько памяти утекло.  
  ```
    ==135348== Memcheck, a memory error detector
    ==135348== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
    ==135348== Using Valgrind-3.15.0 and LibVEX; rerun with -h for copyright info
    ==135348== Command: ./a
    ==135348== 
    ==135348== 
    ==135348== HEAP SUMMARY:
    ==135348==     in use at exit: 4 bytes in 1 blocks
    ==135348==   total heap usage: 2 allocs, 1 frees, 72,708 bytes allocated
    ==135348== 
    ==135348== LEAK SUMMARY:
    ==135348==    definitely lost: 4 bytes in 1 blocks
    ==135348==    indirectly lost: 0 bytes in 0 blocks
    ==135348==      possibly lost: 0 bytes in 0 blocks
    ==135348==    still reachable: 0 bytes in 0 blocks
    ==135348==         suppressed: 0 bytes in 0 blocks
    ==135348== Rerun with --leak-check=full to see details of leaked memory
    ==135348== 
    ==135348== For lists of detected and suppressed errors, rerun with: -s
    ==135348== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)

  ```  
  Скомпилировать с санитайзером можно вот так `g++-10 main.cpp -o a -fsanitize=address`. Вот что он выдаст  
  ```
    =================================================================
    ==136349==ERROR: LeakSanitizer: detected memory leaks

    Direct leak of 4 byte(s) in 1 object(s) allocated from:
        #0 0x7fce088235a7 in operator new(unsigned long) ../../../../src/libsanitizer/asan/asan_new_delete.cpp:99
        #1 0x5633a583225e in main (/home/mirong/CLionProjects/hse/plusi/ut/untitled/a+0x125e)
        #2 0x7fce0821e0b2 in __libc_start_main (/lib/x86_64-linux-gnu/libc.so.6+0x270b2)

    SUMMARY: AddressSanitizer: 4 byte(s) leaked in 1 allocation(s).
  ```
  
* ## Отличия `unique_ptr` от обычного указателя, когда что использовать  
`unique_ptr` это `smart_pointer`. Это некая обёртка над обычным указателем, добовляющая какую-то семантику, кто указателем владеет, кто удаляет, какие-то операции запрещает делать, каие-то делает автоматически. `unique_ptr` сам чистит за собой память, его не нужно удалять руками. При этом его нельзя копировать, только мувать. Получать значение можно так же, как и у обычного указателя.  
Если вы не хотите возиться с удалением памяти, и вам не нужно копирование указателей, то используйте `unique_ptr`.  
* ## Создание `unique_ptr`: какие есть конструкторы, что делает `make_unique`  
Можно просто объявить, тогда это `nullptr`  
```C++
std::unique_ptr<Foo> f;
assert(f == nullptr);
```
Можно собственно записать указатель, с помощью `make_unique`. Под капотом это обычный `new`, только возвращающий сразу `unique_ptr`  
```C++
std::unique_ptr<Foo> f = std::make_unique<Foo>();
```
  * ### Проблема с `make_unique` и приватными конструкторами
  * ### Решения: неработающее с друзьями, работающее с фабричной статической функцией
* ## Невозможность копирования
`unique_ptr` нельзя копировать, только мувать.  
* ## Синтаксис перемещения: в параметры функции, из функции, в/из других переменных (включая поля: `13-211208/02-move-objects/03-move-to-field`), в/из контейнеров
  * ### Когда (не) надо писать `std::move`
* ## `move` как оптимизация для копируемых объектов, автоматическая поддержка `move` у пользовательских структур
  * ### moved-from состояние у объектов

Тесно связано с: функции (как передавать параметры), жизнь объектов.
