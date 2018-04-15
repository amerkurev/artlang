---
author: "Andrey Merkurev"
linktitle: "Отличие reinterpret_cast от приведения типов в стиле С"
title: "Отличие reinterpret_cast от приведения типов в стиле С"
publishDate: 2016-02-16T17:35:00+03:00
date: 2016-02-16T17:35:00+03:00
draft: false
tag: "c++, reinterpret_cast, static_cast, const_cast, c-style"
oldLink: "/article/view/25/"
---


Приведение типов в С\+\+ часто обсуждается на собеседовании. Эта тема особенно актуальна, когда приходится много работать с унаследованным кодом на С. Операторы приведения в С\+\+ давно стали всем привычны: **const_cast**, **static_cast**, **dynamic_cast** и **reinterpret_cast**. И, конечно, никуда не делось  "приведение в старом стиле" \- оригинальный синтаксис С, согласно которому заключенный в скобках тип переменной применяется к выражению **(new_type)** _expression_. 

Сравнивая операторы приведения между собой, программисты без проблем определяют когда и какой оператор нужно применять. О многом "подсказывают" сами названия этих операторов. **reinterpret_cast **при этом, всегда сравнивают с **приведением типов в стиле С**. "_Не гарантирует переносимость кода, опасно и нежелательно_". Всё это правильно, но вопрос в чём же тогда разница? Что делает **reinterpret_cast**, а что приведение типов в стиле С ? Частый ответ \- это одно и тоже! И тут начинаются дополнительные вопросы...

### **reinterpret_cast < new_type >** _(expression)_

Это прямое указание компилятору обращаться с некоторой последовательностью битов, являющихся результатом выражения _(expression)_, так будто это объект типа **new_type**. Можно, например, привести целое к указателю, или один указатель к иному произвольному указателю. 

Понято, что если такая интерпретация битов не верна, нас ждут проблемы. "В общем случае, результат операции **reinterpret_cast **гарантировано приемлем для использования лишь тогда, когда преобразуемое значение соответствует целевому типу" (Страуструп). Особенно коварны случаи использования **reinterpret_cast **при наследовании классов. Небольшой пример:
```
#include <iostream>

struct A {
    int a;
};

struct B {
    int b;
};

struct C : public A, public B {
};

int main() {
    C c;
    c.a = 1;
    c.b = 2;
    
    std::cout << reinterpret_cast<B*>(&c)->b << std::endl;  // result 1 -> BAD!
    std::cout << static_cast<B*>(&c)->b << std::endl;       // result 2 -> OK!
}
```
В этом примере корректно отработал только **static_cast.** Хотя **reinterpret_cast **и выдал результат (программа собралась и успешно выполнилась), но этот результат далеко не то, что было нужно нам. Дело в том, что **static_cast **осуществляет правильную работу с адресами, в то время как **reinterpret_cast **просто интерпретирует указатель, так как "приказывает" программист, не меняя его значения. Именно этим и опасен **reinterpret_cast**, он "всё воспринимает на веру", не проверяя, что мы хотим на самом деле. А как же тогда поведёт себя приведение в "старом стиле" ? Рассмотрим.

### **Приведение типов в стиле С **

Прежде чем, продолжить, посмотрите на тот же пример, но уже с приведением в стиле С.
```
#include <iostream>

struct A {
    int a;
};

struct B {
    int b;
};

struct C : public A, public B {
};

int main() {
    C c;
    c.a = 1;
    c.b = 2;
    
    std::cout << ((B*)(&c))->b << std::endl;           // result 2 -> OK!
    std::cout << static_cast<B*>(&c)->b << std::endl;  // result 2 -> OK!
}
```
Всё отработало так, как и положено: приведение типов в стиле С справилось с поставленной задачей! Но почему? Случайность? А если нет, то как тогда можно утверждать, что это одно и тоже с **reinterpret_cast**. Вот еще один пример:
```
#include <iostream>

int main() {
    int i = 1;
    const int* p = &i;
    
    std::cout << *((int*)(p)) << std::endl; // OK!
    std::cout << *(reinterpret_cast<int*>(p)) << std::endl;  // Error!
}
```
Этот пример даст нам ошибку на этапе компиляции:
```
reinterpret_cast from 'const int *' to 'int *' casts away qualifiers
```
Получается, что у приведения типов в стиле С нет проблем справиться с указателем на константу. А вот **reinterpret_cast **в той же ситуации "подвёл", и не справился с задачей. Впрочем, помочь ему мог бы оператор **const_cast**.
```
int main() {
    int i = 1;
    const int* p = &i;
    
    std::cout << *(reinterpret_cast<int*>(const_cast<int*>(p)));  // OK!
}
```
Все эти примеры наглядно демонстрируют что, **приведение типов в стиле С** и оператор **reinterpret_cast **\- это далеко не одно и тоже! Почему же "старое приведение" работает в некоторых случаях лучше, чем **reinterpret_cast **? Чтобы в этом разобраться, нужно представлять как компилятор интерпретирует "старое приведение" типов, когда встречает его в коде.

Если компилятор встретил приведение типов в стиле С, то он делает ряд попыток задействовать один или несколько имеющихся у него в распоряжении операторов приведения типов. Все попытки компилятор выполняет в строгом порядке, от "наиболее безопасных" к "самым рискованным". Если очередная попытка позволена с точки зрения применения операторов приведения, то поиск прекращается. Если нет, то рано или поздно, компилятор дойдет до **reinterpret_cast**. Последовательность шагов компилятора:

<ol><li>сначала пробуем применить оператор&nbsp;<strong>const_cast</strong>&lt;new_type&gt;(expression).&nbsp;Возможно этого будет достаточно. Если нет, то шаг 2.</li><li><strong>static_cast</strong>&lt;new_type&gt;(expression). Именно благодаря этому шагу, приведение в стиле С отработало правильно, а&nbsp;<strong>reinterpret_cast</strong> - нет, в приведенных примерах.</li><li>А теперь комбинация из&nbsp;<strong>static_cast</strong>, который следует за&nbsp;<strong>const_cast</strong>. Если и это не помогло, то...</li><li><strong>reinterpret_cast</strong>&lt;new_type&gt;(expression). Но как было видно из последних примеров, даже&nbsp;<strong>reinterpret_cast </strong>не в силах справиться с квалификатором <strong>const</strong>. А приведение в стиле С, работает всегда - благодаря шагу №5 (последнему).</li><li><strong>reinterpret_cast</strong>, который следует за&nbsp;<strong>const_cast</strong> (последний пример).&nbsp;На этом шаге компилятор уже точно, сделает то, что Вы просите. Чего бы это ни стоило.</li></ol>

Получается, что приведение типов в стиле С \- это "оборотень": в каких-то случаях оно ведет себя как **static_cast**, в каких-то как **reinterpret_cast**, а иногда и как **const_cast**. 


### **Вывод**

После этого краткого обзора, у Вас могло появиться стойкое желание использовать везде "старое доброе приведение" в стиле С, и пусть компилятор сам разберется, вместо того, чтобы случайно "посадить" где-то ошибку с **reinterpret_cast**. Но это не совсем правильный вывод.

Операторы приведения типов позволят другим программистам лучше осознать намерения Ваших действий. Вы сами скажите себе "спасибо" через некоторое время. Правильный вывод: проектировать так, чтобы не использовать приведение типов вообще, а если и применять, то только то, что действительно нужно. "Ювелирные" приведения часто необходимы, но если начать всюду стрелять из пушки под названием **C-style cast**, то можно случайно попасть себе же в ногу.