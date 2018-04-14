---
author: "Andrey Merkurev"
linktitle: "Про size() у std::list, в чем сложность?"
title: "Про size() у std::list, в чем сложность?"
publishDate: 2014-11-06T11:55:00+03:00
date: 2014-11-06T11:55:00+03:00
draft: false
tag: "c++, list, merge, splice, remove, reverse, sort, unique, stl"
oldLink: "/article/view/5/"
---


Практически все контейнеры из стандартной библиотеки STL реализуют метод **size**. Всем известно, что данная функция-член возвращает количество элементов в контейнере. Однако,  сложность такой операции для известного контейнера **std::list **не определена однозначно. В реализации STL от Microsoft сложность операции size равна O(1), то есть константная, и не зависит от количества элементов. Напротив, в GCC сложность операции size равна O(N) - линейная, что существенно ограничивает возможности её применения, особенно там, где важна производительность. Ведь каждый раз при вызове size, Вы фактически будете инициировать полный проход по всему списку. В чем же причина такой неоднозначности?

_Материал статьи основан на публикации Howard Hinnant: _

[http://howardhinnant.github.io/On\_list\_size.html](http://howardhinnant.github.io/On_list_size.html)

_Howard Hinnant работает в компании Apple, и является членом комитета по стандартизации C++ от компании Apple. Также он занимается разработкой библиотеки Boost._

До стандарта C++11 сложность метода size у списков описывалась, как "от константной до линейной". То есть, на деле все зависило от конкретной реализации контейнера list. Чтобы сделать сложность константной, внутри контейнера помещали счетчик элементов и изменяли его при добавлении или удалении элементов. Тогда реализация size, например у Microsoft, выглядела так:
```
size_t size() const
{	
    // Актуальное число всех элементов всегда хранится в _Mysize, поэтому O(1)
    return (this->_Mysize); 
}
```
В GCC не создавали дополнительных счетчиков и size в этом случае реализовывался так:
```
size_t size() const
{ 
    // Пересчитываем все элементы, одни за другим, поэтому имеем O(N)
    return std::distance(begin(), end());  
}
```
Определенно, реализация от Microsoft выглядит предпочтительнее. Ведь, если Вы используете gcc, то вызова size нужно практически избегать, так как он крайне неэффективен. Но в пользу линейной сложности для size всегда приводился "весомый" аргумент.

### Недостатки size c O(1)

Список отличается наличием эффективных методов для работы с ним. Эти методы схожи со стандартными алгоритмами, но при работе со списками в них никогда не применяется непосредственное копирование элементов. Вместо этого просто изменяются указатели на предыдущий и следующий элементы. Поэтому у списка есть особые методы: **merge**, **splice**, **remove**, **reverse**, **sort**, **unique**.  

Операции splice ("сращивание" двух списков) имеет сложность O(1), так как просто изменяет значения указателей. Но, требование от size иметь сложность равную O(1) "портит" одну из версий splice, а именно эту:
```
splice(iterator position, list& x, iterator first, iterator last);
```
Если size должен иметь сложность O(1), и this != &x (соединяются разные списки), тогда данный метод должен иметь линейную сложность O(N), так как придется снова пересчитать количество элементов в объединенном списке, чтобы обеспечить сложность O(1) для size. Если бы сложность size могла быть линейной, то эта версия splice имела бы сложность O(1), как и её другие перегруженные версии. Это и есть, тот самый аргумент в пользу медленного size:

**size O(1) приводит к splice O(N) **

И он вполне справедлив, ведь каждый контейнер ценится за тот набор эффективных операций, которые он предоставляет и жаль "терять" одну из таких эффективных операций.  

Но как считает Howard Hinnant этот довод сильно преувеличен. И вот почему: рассмотрим 5 возможных ситуаций соединения (splice situations) списков: соединение с самим собой или с другим списком одного, нескольких или всех элементов сразу. При условии, что size имеет сложность O(1), только один из пяти вариантов "сращивания" списков потеряет в эффективности:
  

<table border="1" cellspacing="1" cellpadding="1" style="width:550px"><tbody><tr><td><br></td><td>splice с самим собой</td><td>splice с другим списком</td></tr><tr><td>один элемент</td><td>O(1)</td><td>O(1)</td></tr><tr><td>несколько элементов</td><td>O(1)</td><td>O(some) - distance(first, last)</td></tr><tr><td>все элементы</td><td>Not valid</td><td>O(1)</td></tr></tbody></table>

Для того, чтобы обеспечить O(1) для size нужно посчитать те несколько элементов (some), которые будут перенесены в результирующий список (O(some)).

### Преимущества size c O(1)

Тем не менее, если size будет иметь сложность O(1), это позволит задействовать оптимизацию для следующих методов:
```
list& operator=(const list&);
iterator assign(iterator, iterator); 
void assign(size_type, const T&);
```
В случае, когда размер списка уменьшается, использование константного size поможет определить какой вариант прохода будет эффективнее: от начала списка до начала удаляемого диапазона, или от конца списка до начала удаляемого диапазона: 
```
// Метод может работать лучше, если мы удаляем несколько элементов в конце или
// оставляем несколько элементов в начале

void resize(size_type, T);
```
Операции сравнения списков также могут быть оптимизированы, если вначале сравнить размеры списков, а только потом, при условии равенства результатов size, начинать по-элементное сравнение списков: 
```
// Будут работать эффективнее, 
// так как если l1.size() != l2.size() => быстрый вывод, что списки не равны

bool operator==(const list& l1, const list& l2); 
bool operator!=(const list& l1, const list& l2); 
```
Доводы в пользу константной сложности метода size видимо пересилили и, начиная со стандарта C++11, разработчикам STL рекомендуется для size обеспечивать сложность равную O(1). Но Вы должны иметь ввиду, что на практике size может по-прежнему иметь сложность O(N). Например, в gcc-4.9.1 реализация по-прежнему такая:
```
/**  Returns the number of elements in the %list.  */
size_type size() const _GLIBCXX_NOEXCEPT
{ 
    return std::distance(begin(), end()); // медленно!
}
```
Howard Hinnant в своей статье приводит несколько мест из библиотеки boost, где отмечает потенциальное ухудщение производительности, в случае использования "медленного" (O(N)) size.

Howard Hinnant также считает, что если контейнер не может обеспечить для size сложность O(1), то такой контейнер не должен вовсе иметь метод size. А его размер, при необходимости, должен высчитываться, например, с использованием алгоритма sdt::distance. Это будет явно указывать на линейную сложность. Но это требование применимо лишь к новым контейнерам (таким как std::forward_list). Поэтому, если Вы используете std::list и компилируете код разными компиляторами, Вам следует учитывать сложность метода size в разных реализациях STL, чтобы иметь действительное представление об эффективности Вашей работы со std::list.

**P.S.**: Если Вам нужно проверить условие, что контейнер не пуст, всегда используйте метод empty(), а не условие size() > 0. Empty всегда будет или быстрее, или таким же по скорости, как size.