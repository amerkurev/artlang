---
author: "Andrey Merkurev"
linktitle: "В C++17 могут появиться новые глобальные функции"
title: "В C++17 могут появиться новые глобальные функции: std::size, std::data и std::empty"
publishDate: 2015-01-23T21:58:00+03:00
date: 2015-01-23T21:58:00+03:00
draft: false
tag: "c++, std::size, std::data и std::empty, c++17, constexpr"
oldLink: "/article/view/11/"
---


Такие функции-члены контейнеров как: size, empty, front, back и data - хорошо знакомы всем C++ программистам. Они давно являются незыблемым инструментом при разработке. Но, оказывается, в комитет поступило предложение, которое призвано улучшить работу с этими функциями.

Эта статья основана, и, фактически является переводом вот этого [предложения N4017](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4017.htm), которое поступило в комитет в мае прошлого года (2014). Автор Riccardo Marcangelo - кто это, узнать не удалось. Что же он предлагает?

Рикардо, говорит: давайте добавим в библиотеку глобальные функции: std::size, std::empty, std::front, std::back, и std::data, которые сейчас существуют в виде функций-членов контейнеров. Включение этих функций обеспечит преимущества в отношении безопасности, эффективности и универсальности. Все подробности ниже (статья очень короткая).

Обычной задачей, особенно при работе с устаревшим кодом, является определение количества элементов в "сишном статическом" массиве, иначе говоря, длины массива. Обычно, это достигается за счет использования оператора sizeof, который применяется ко всему массиву, и делением результата на размер одного элемента массива:

```
#include <iostream>

int main()
{
    int a[100];
    std::cout << sizeof(a)/sizeof(*a); // Количество элементов в массиве
}
```
Для удобства, подобный функционал часто "оборачивают" в макрос и называют как "ARRAYSIZE" или "countof":
```
#define ARRAYSIZE(_array) (sizeof(_array) / sizeof(_array[0]))
```
Проблема заключается в том, что такой макрос не безопасен. Если вместо статического массива будет передан указатель, код успешно скомпилируется, но работать правильно не будет. При этом мы даже не получим предупреждений от компилятора, не говоря уже об ошибках. У Microsoft есть более "хитрая" версия макроса \_countof, которая "умеет" выдавать ошибки на этапе компиляции, если ей будет передан указатель:
```
// Пример взят из MSDN-а, он всё наглядно показывает

#define _UNICODE
#include <stdio.h>
#include <stdlib.h>
#include <tchar.h>

int main( void )
{
   _TCHAR arr[20], *p;
   printf( "sizeof(arr) = %d bytes\n", sizeof(arr) ); // 40 bytes
   printf( "_countof(arr) = %d elements\n", _countof(arr) ); // 20 elements
    
   // In C++, the following line would generate a compile-time error:
   // printf( "%d\n", _countof(p) ); // error C2784 (because p is a pointer)

   _tcscpy_s( arr, _countof(arr), _T("a string") );
   // unlike sizeof, _countof works here for both narrow- and 
   // wide-character strings
}
```
Используя возможности C++11, мы можем заменить небезопасный макрос функцией, со спецификатором **constexpr. **​Constexpr позволяет гарантировать, что функция возвращает константу времени компиляции:​
```
// Такая функция обеспечивает замену старого макроса

template <class T, std::size_t N>
constexpr std::size_t size(const T (&array)[N]) noexcept
{
    return N;
}
```
Что-то похожее сделано и у Microsoft, но они "обошлись" без constexpr:
```
template <typename _CountofType, size_t _SizeOfArray>
char (*__countof_helper(_CountofType (&_Array)[_SizeOfArray]))[_SizeOfArray];

#define _countof(_Array) (sizeof(*__countof_helper(_Array)) + 0)
```
Сейчас, у нас в распоряжении есть алгоритм std::distance, с помощью которого, мы можем единообразно узнавать размер как контейнера, так и статического массива:
```
#include <iostream>
#include <algorithm>

int main()
{
    int a[10];
    std::cout << std::distance(std::begin(a), std::end(a));
}
```
Однако, как заметил Stephan T. Lavavej, std::distance - это нагруженный (verbose) и неэффективный способ для многих контейнеров. Дело в том, что многие контейнеры определяют функцию-член size, которая имеет константную сложность O(1), то есть выполняется за постоянное время вне зависимости от количества элементов. Напротив, std::distance всегда будет иметь линейную сложность для контейнеров, итераторы которых слабее, чем итераторы произвольного доступа.

К тому же алгоритм std::distance не использует спецификатор **constexpr**.

Интересно, что Александер Степанов (создатель STL), в своих ["Заметках о программировании",](http://www.stepanovpapers.com/notes.pdf) пишет следующее: "Я сделал size функцией-членом в попытке угодить комитету по стандартизации. Я знал, что begin, end и size должны быть глобальными функциями, но не был готов к еще одному спору с членами комитета". К счастью, в C++11 включены глобальные функции std::begin() и std::end(), однако, жаль, что нет глобального std::size.

_Было бы здорово добавить глобальную функцию std::size(), которую можно было бы использовать, как с контейнерами, так и со статическими массивами. Это позволило бы писать более обобщенны код. Также, было бы полезно иметь и другие глобальные функции, такие как: std::empty(), std::front(), std::back(), и std::data()._

Тогда мы смогли бы писать примерно так:
```
// Пример целиком взят из оригинала

int builtin_one [10] = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
std::vector<int> vec (5, 13);
 
//non-member size
int builtin_two [std::size(builtin_one)];
std::array<int, std::size(builtin_one)> arr;
               
std::cout << std::size(builtin_one) << '\n';
std::cout << std::size(builtin_two) << '\n';
std::cout << std::size(arr) << '\n';
std::cout << std::size(vec) << '\n';         
           
//non-member front/back
//allow the result to be used as an lvalue
std::front(builtin_two) = 53;
std::back(vec) = 11;
std::cout << std::front(builtin_two) << '\n';
std::cout << std::back(vec) << '\n';
           
//non-member empty
std::cout << std::boolalpha << "std::empty(builtin_one): " << std::empty(builtin_one) << '\n';
std::cout << "std::empty(vec): " << std::empty(vec) << '\n';
           
//non-member data
std::cout << "std::data(builtin_one): " << std::data(builtin_one) << '\n';
std::cout << "std::data(vec): " << std::data(vec) << '\n';   
```
Используя C++11, можно получить размер массива альтернативным способом:
```
int array [10];
auto array_size = std::extent<decltype(array)>::value;
```
Недостатком такого подхода является то, что мы теряем возможность обобщить его одновременно и для массивов, и контейнеров STL.

Далее в статье затрагивается вопрос реализации, который я опускаю. Кстати, предложение уже прошло две доработки, и, в рамках [одной из них](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4155.htm), из рассмотрения были убраны функции front, back.

Мы не скоро увидим то, о чем здесь рассказывается, но все-таки это предложение мне показалось интересным и простым одновременно. Поэтому решил поделиться.