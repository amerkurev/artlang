---
author: "Andrey Merkurev"
linktitle: "Vector в C++: основные приемы эффективного использования"
title: "Vector в C++: основные приемы эффективного использования"
publishDate: 2014-10-24T23:36:00+03:00
date: 2014-10-24T23:36:00+03:00
draft: false
tag: "c++, allocator, operator, stl, new, reserve, vector, emplace_back"
oldLink: "/article/view/4/"
---


Данная статья рассчитана на неискушенного программиста C++. С другой стороны любой, кто программирует на C++ обязан знать то, что описывается ниже. Речь пойдет об одном из самых популярных контейнеров в STL. А именно, о векторе (std::vector). С него, обычно, начинается описание контейнеров в книгах по языку C++ и STL. Считается, что это наиболее простой контейнер, с понятным интерфейсом и предсказуемым временем операций. Однако, за описанием интерфейса часто "теряются" основные правила при работе с вектором, которые могут быть не так очевидны  для начинающего программиста. Но, обо всем попорядку.

![](/images/vector-1.png)


### **Внутреннее устройство вектора**

Джосаттис в своей книге описывает вектор так: "Этот контейнер моделирует динамический массив. Таким образом, вектор \- это абстракция, управляющая своими элементами в стиле динамического массива из языка C. Однако в стандарте не указано, что реализация вектора использует динамический массив. Скорее это следует из ограничений и спецификаций сложности его операций". 

Вектор представляет собой шаблонный класс в пространстве имен std:
```
namespace std {
    template < typename T, 
               typename Allocator = allocator<T> >
    class vector;
}
```
Второй необязательный шаблонный параметр задает модель памяти. По умолчанию используется шаблонный класс allocator. Этот класс предоставляет стандартная библиотека C++, и именно он отвечает за выделение и освобождение памяти. Если вектору в качестве второго параметра передать какой-либо другой распределитель памяти (allocator), то вектор, возможно, уже будет основан не на C-массиве. Однако, обычно второй параметр остается заданным по умолчанию, поэтому рассмотрим подробнее стандартный распределитель памяти - std::allocator. Он, кстати, используется по умолчанию для всех контейнеров библиотеки STL. 

Вектору (и всем другим контейнерам STL) от аллокатора нужно прежде всего, чтобы он выделял и освобождал некоторую область памяти, в тот момент когда это требуется контейнеру, Стандартный аллокатор делает это так:
```
template<class T>
class allocator
{
public:
    typedef T value_type;
    
    typedef value_type *pointer;
    typedef const value_type *const_pointer
        
    typedef size_t size_type;
    
    pointer allocate(size_type _Count)  // Выделяем память для _Count элементов
    {                                   // типа value_type  
        void *_Ptr = 0;

        if (_Count == 0)  // Если ничего не запросили, то ничего и не делаем
            ;
        else if (((size_t)(-1) / sizeof (value_type) < _Count)
            || (_Ptr = ::operator new(_Count * sizeof (value_type))) == 0)
        {
            throw bad_alloc();  // Выделение памяти не удалось
        }   
        return ((pointer)_Ptr);
    }
    
    void deallocate(pointer _Ptr, size_type)
    {   
        ::operator delete(_Ptr);  // Освобождение памяти
    }
    
    // Остальная реализация не приводится, чтобы сохранить наглядность примера
};
```
Так аллокатор реализован у Microsoft (MSVC), и также он реализован в GCC. Нужно понимать, что **operator new** отличается от просто **new**. Вызов **new** (который все мы используем) на самом деле разбивается на два этапа, если так можно выразиться: сначала вызывается **operator new**, который возвращает указатель на некоторую выделенную область памяти, а затем вызывается конструктор, который создает объект в этой области. Вызвать конструктор напрямую невозможно, однако с помощью синтаксиса размещения можно заставить компилятор вызвать коструктор. Следующие два примера создания foo1 и foo2 идентичны:
```
#include <new>

class Foo
{
public: 
    Foo() {}
};

int main(int argc, char** argv) 
{
    // Вот так мы все привыкли создавать объект в "куче":
    Foo *foo1 = new Foo(); // Выделение памяти + Вызов конструктора

    // А вот какие вызовы происходят на самом деле:
    void *ptr = operator new(sizeof(Foo)); // Выделение памяти
    Foo *foo2 = ::new (ptr) Foo(); // Вызов конструктора, синтаксис размещения
    
    return 0;
}
```
Широко распространно заблуждение, что применение оператора new (первый случай в примере) подразумевает необходимость работать с кучей (heap). Вызов new, лишь означает, что будет вызвана функция operator new и эта функция возвратит указатель на некоторую область памяти, Стандартные operator new и operator delete действительно работают с кучей, но члены operator new и operator delete могут делать всё, что угодно! Нет ограничения на то, где будет выделена область памяти. Но вернемся к вектору.

После того, как память будет выделена, она "переходит" под управление вектора. До стандарта C++11 все аллокаторы должны были быть максимум "stateless", т. е. не хранить никаких состояний. Поэтому все состояния (переменные) хранит вектор. Как Вы думаете, сколько нужно переменных вектору, чтобы реализовать весь свой интерфейс? Ответ - три. Трех указателей достаточно, чтобы вектор смог управлять памятью и обеспечил весь свой функционал:

1.  первый (first) указатель указывает на начало выделенной области памяти
2.  второй (last) указатель указывает на позицию следующую за последним элементом, хранящимся в выделенной области памяти
3.  и третий (end) указывает на "конец" выделенной области памяти

Они очень просто инициализируются сразу после того, как аллокатор выделит память:
```
// _Capacity - Количество элементов, которое может быть размещено в памяти вектора
first = Getal().allocate(_Capacity); // Getal() возвращает ссылку на аллокатор
last = first;  // В начале вектор пуст, затем last начинает смещаться к end
end = first + _Capacity; // Граница выделенного участка памяти, пересекать её нельзя!
```
Посмотрите теперь, как лаконично и кратко реализуются некоторые функции-члены вектора:
```
size_t capacity() const
{   // return current length of allocated storage
    return (end - first);
}

size_t size() const
{   // return length of sequence
    return (last - first);
}

bool empty() const
{   // test if sequence is empty
    return (first == last);
}

const_reference at(size_t pos) const
{   // subscript nonmutable sequence with checking
    if (last - first <= pos) throw out_of_range();
    
    return (*(first + pos));
}

const_reference operator[](size_t pos) const
{   // subscript nonmutable sequence
    return (*(first + pos));
}
// Сплошная арифметика указателей...
```
Когда программист добавляет в конец вектора новый элемент происходит примерно следующее:
```
void push_back(const value_type& val)
{   // insert element at end
    
    if (last == end) {      
    // Если память закончилась, 
    // то нужно запросить новую область, больше предыдущей    
        
        // размер текущей области памяти
        size_t _Capacity = capacity();
        
        // требуемый минимальный размер новой области
        size_t _Count = size() + 1;     
        
        if (max_size() - _Capacity / 2 < _Capacity) {
            
            // новая область больше на 50% предыдущей
            _Capacity = _Capacity + _Capacity / 2;  
        }
        
        if (_Capacity < size() + 1)  {
            
            // Если нет возможности увеличить на 50%, то
            // пробуем выделить минимально необходимый размер
            _Capacity = _Count;      
        }
        
        _Reallocate(_Capacity); 
        // Перераспределение памяти, значения fisrt, last и end
        // будут изменены! Предыдущий участок памяти будет освобожден. 
        // В новый участок будут скопированы все данные из предыдущего. 
        // Это не быстрый процесс...
    } 
    
    // Вызов копирующего конструктора, синтаксис размещения
    ::new ((void *)last) value_type(val);
    
    ++last; // Сдвигаем указатель с учетом вставленного элемента
}
```
Данный код взят из исходников STL MSVC, но он не сильно отличается от аналогичного кода в GCC. Код был несколько упрощен, чтобы суть алгоритма не была "размыта" и оставалась ясной. 

Итак. Вектор гарантирует (по стандарту), что вставка в конец происходит очень быстро за одно и тоже время. Однако если мы попадаем под условие last == end (если выделенная память закончилась) то время вставки в конец сильно возрастает. То на сколько затянется процесс трудно спрогнозировать и это в большей степени зависит от конструкторов копирования и деструкторов элементов, находящихся в вектре. Так как они будут вызваны для каждого существующего элемента в векторе при перераспределении памяти.

  

### **Приемы эффективного использования**

В предыдущем примере хорошо видно, как реализация вектора "пытается бороться" с неэффективным использование памяти. Проблема возникает, когда память, выделенная аллокатором, заканчивается. В этот момент вектор запрашивает у аллокатора новый участок памяти, больше предыдущего, которой смог бы разместить все элементы уже содержащиеся в векторе, а также новые. Для этого повторно выделяется память из "кучи". Операция эта достаточно медленная, и к тому же блокирующая в общем случае (heap lock - это "узкое место" многопоточных приложений). Таких операций должно быть как можно меньше.

Поэтому вектор запрашивает память "про запас", чтобы не инициировать новое перераспределение памяти при добавлении следующего элемента. Если бы вектор на каждый push_back вызывал перераспределение памяти, он был бы очень медленным. Вместо этого, когда память оказывается исчерпана, вектор запрашивает на 50% больше памяти, чем у него было. Так он уменьшает количество повторных обращений к аллокатору и, тем самым, перераспределений памяти.

При перераспределении памяти, в общем случае, вызываются копирующие конструкторы элементов вектора, так как из "старой" памяти их нужно корректно перенести в новую выделенную область, А вслед за конструкторами вызываются деструкторы элементов, так как из "старой" памяти их нужно правильно удалить. Если в векторе хрянятся POD данные, то конечно будет задействована оптимизация при их копировании в новую область, однако это скорее частный случай, нежели общая практика.

Рассмотрим пример, в котором мы заполним вектор одним миллионом объектов класса Foo. Будем считать, что к Foo не применяется оптимизация при копировании:
```
class Foo
{
public: 
    
    explicit Foo(long long) {}
    Foo(const Foo&) {}
    ~Foo() {}
};


int main(int argc, char** argv) 
{
    std::vector<Foo> vec;

    for (long long ii = 0; ii < 1000000; ++ii) {
        vec.push_back(Foo(ii));
    }
    return 0;
}
```
По выходу из блока for, оказалось что: 

*   копирующий конструктор был вызван более 3 миллионов раз
*   деструктор был вызван более 3 миллионов раз
*   перераспределение памяти произошло 35 раз (в начале емкость вектора была равна 0)

Ни одна из этих миллионов операций нам не была нужна. Поэтому попробуем избавиться от них.

### reserve(_num_)

reserve - это функция-член вектора, которая увеличивает его емкость, то есть принудительно запрашивает у аллокатора область памяти такого размера, чтобы вектор, при желании, мог разместить в ней num элементов. Вызывать её стоит сразу после создания объекта вектора, пока в нем еще нет элементов. Допустим Вы можете не знать точного числа элементов, но Вы как минимум можете предположить это значение, затем накиньте еще 50% (если нет проблем с памятью) и это значение передайте reserve. Вызвать reserve имеет смысл практически всегда, даже если число элементов не большое. Так Вы поможете вектору избежать множества лишних шагов. Если в примере, сразу после создания объекта вектора, вызвать reserve:
```
std::vector<Foo> vec;
vec.reserve(1000000); // Сразу зарезервируем место под миллион элементов
```
то по выходу из блока for, окажется что: 

*   копирующий конструктор был вызван ровно 1 миллион раз
*   деструктор был вызван ровно 1 миллион раз
*   перераспределение памяти не происходило ни разу

Отличный результат! Миллионы лишних вызовов удалось избежать, к тому же перераспределение памяти ни разу не произошло! А всё потому что вектору хватило памяти разместить все наши элементы. Всегда помните об этом, и вызывайте reserve, после создания вектора. Это полезно еще и потому, что стандатрный механизм выделения "про запас" (на 50% больше, чем было) иногда может привести к избыточному выделению памяти. Много лишнего, это тоже плохо. Лучше контролируйте этот процесс сами.

### emplace_back(args)

Идем дальше. Порой удается избежать вообще вызовов копирующего конструктора при добавлении нового элемента. Начиная со стандарта C++11 появилась функция-член emplace\_back, которая конструирует элемент прямо на месте, где его и предполагалось разместить. При этом не вызывается ни копирующий конструктор, ни перемещающий. Аргументы, переданные функции emplace\_back, точно также передаются затем конструктору элемента. Перепишем добавление элементов в примере, задействова emplace_back:
```
std::vector<Foo> vec;
vec.reserve(1000000); // Сразу зарезервируем место под миллион элементов

for (long long ii = 0; ii < 1000000; ++ii) {
    
    // Передаем аргументы так, будто вызываем обычный конструктор Foo
    vec.emplace_back(ii); 
}
```
По выходу из блока for, оказалось что: 

*   копирующий конструктор не вызывался ни разу
*   деструктор не вызывался ни разу 
*   перераспределение памяти не происходило ни разу

Чтобы разместить один миллион элементов Foo, потребовалось только вызвать миллион раз конструктор данного типа, что вполне логично. Напомним, что до предпринятых действий, было более шести миллионов ненужных вызовов и несколько перераспределений памяти.

### POD

Еще одним способом оптимизировать работу с вектором, является использование POD ("plain old data") типов в качестве элементов. Так как вектор управляет непрерывным участком памяти, он может применять тривиальные функции копирования, наподобие memcpy. Новый стандарт C++11 несколько расширил понятие POD. Обязательно ознакомьтесь с этими простыми структурами данных, и смело используйте их в качестве элеметов вектора, так как для них будет применяться оптимизация при копировании. 

### Добавляйте в конец и меньше ищите

Если Вам часто нужно что-то искать в векторе, то возможно стоит подумать о выборе другого контейнера. Поиск по несортированной последовательности крайне неэффективен, но еще более неэффективно пытаться держать вектор отсортированным. Например, вызвав std::sort Вы инициируете неизвестное Вам количество вызовов копирующего конструктора и такое же количество деструкторов. В худшем случае это число может стремиться к общему числу элементов в векторе.

Что же касается вставки элемента в вектор, то здесь можно легко подсчитать чего это будет стоить. Для всех элементов, находящихся за позицией вставки, будут вызваны по разу копирующий конструктор и деструктор. На нашем примере, если мы решим вставить новый элемент вот таким образом:
```
// ... после цикла for
vec.insert(vec.begin() + 500000, Foo(1));
```
то мы получим:

*   500000 ненужных нам вызовов копирующего конструктора
*   500000 ненужных нам вызовов деструктора

Вот такая плата. Поэтому, если Вы часто вставляете элементы не в конец вектора, возможно стоит подумать о выборе другого контейнера.

### **Итог**

Вектор \- это отличный контейнер, который подходит для размещения очень большого количества элементов. Он умеет резервировать память, что отличает его от остальных контейнеров, и это нужно использовать. POD данные - идеальные кандидаты на элементы вектора, так как к ним вектор применяет простые и быстрые методы копирования. Начиная со стандарта C++11, мы можем конструировать объекты непосредственно на том месте, где они будут храниться, если будем использовать emplace_back. Это экономит вызовы копирующих конструкторов.  

Вектор плохо подходит для поиска и вставкам элементов в произвольное место (эффективно добавляются элементы только в конец). Помните, при грамотном использовании вектора, Вы всегда получите быстродействие сравнивнимое с кодом на чистом C, при этом у Вас будет удобный интерфейс и автоматическое управление памятью, в придачу. Такого Вам не сможет дать ни один язык - одновременно и быстродействие, и удобство программирования. Если же быстродействие Вам не нужно - то, возможно, Вам не нужен и C++.