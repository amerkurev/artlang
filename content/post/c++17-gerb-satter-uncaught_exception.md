---
author: "Andrey Merkurev"
linktitle: "Герб Саттер и \"бесполезная\" функция uncaught_exception"
title: "C++17: Герб Саттер и \"бесполезная\" функция uncaught_exception"
publishDate: 2015-04-07T00:25:00+03:00
date: 2015-04-07T00:25:00+03:00
draft: false
tag: "c++, c++17, uncaught_exception"
oldLink: "/article/view/12/"
---


Все мы знаем, а многие даже читали, книгу Герба Саттера "Решение сложных задач на С++", которая появилась благодаря известным публикациям из серии под названием "Guru of the Week". Одна из задач в этой книге была посвящена std::uncaught\_exception и звучала она так:

GotW \#47: "_Что собой представляет стандартная функция std::uncaught\_exception и когда она должна использоваться?_"

Ответ дан в книге Саттера еще в далеком 2002 году. Однако, комитет решил вернуться к обсуждению этой "редкой" функции в 2013. Что же решили изменить и для чего все-таки нужна эта функция, я постараюсь "за 5 минут" рассказать в данном посте. Уделите время, и Вы также узнаете про декларативный подход в обработке ошибок, предложенный Александреску.

### **Бесполезная функция**

Стандартная функция std::uncaught\_exception() позволяет понять, не является ли в настоящее время активным какое-то исключение. Эта функция возвращает булевое значение, которое равно _**true**_ в том случае, если в момент вызова std::uncaught\_exception() происходит раскрутка стека в данном потоке.

Как часто она применяется на практике? Скорее всего очень редко, возможно, что и никогда. В своей книге Саттер, после нескольких показательных примеров, говорит следующее:

_"К сожалению, я не знаю ни одного полезного и безопасного применения функции std::uncaught\_exception(). Мой совет: не используйте её!" [GotW #47](http://www.gotw.ca/gotw/047.htm)_

Казалось бы, что после таких слов про эту функцию можно было забыть навсегда. Однако, в 2013 году Саттер вновь возвращается к ней, и даже пишет предложение в комитет под номером [N3614](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3614.pdf) (и уточнение [N4152](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4152.pdf)), в котором предлагается эту функцию заменить... Очень показательно то, что сам Саттер "не забывал" про std::uncaught\_exception(), хотя и советовал не использовать  эту функцию. Зачем же снова обсуждать её, и пытаться что-то улучшить в новом стандарте С++17?

### **ScopeGuard**

Функция std::uncaught\_exception снова попала в поле зрения во многом благодаря Андрею Александреску. Саттер пишет, что именно его примеры послужили главным мотивом к пересмотру возможностей uncaught\_exception. Речь идет о реализации класса **ScopeGuard**, который предложил Александреску. Вот [ссылка](https://vimeo.com/97329153) на его лекцию, которая называется "Error Handling in C++". Она довольно большая и подробная, я же постараюсь "в двух словах" рассказать, зачем нужен ScopeGuard. 

**ScopeGuard** \- это шаблон класс, который позволяет выполнить какие-либо действия в рамках определенной области, в том случае если, в данной области произошло исключение, и в случае, если исключения не произошло. Если Вы знакомы с языком Python или Java, то знаете про оператор **finally**, который позволяет выполнить блок инструкций в любом случае, было ли исключение, или нет. Язык С\+\+ не поддерживает finally, однако у нас есть идиома RAII. Деструктор локальной переменной будет вызван при выходе её из области видимости, и в случае возникновения исключения. А значит мы сможем сделать те действия, которые "поместили" бы в блок finally, будь он в нашем арсенале. Но всё не так просто...

Что если действия, которые требуется выполнить, различны для случая, когда было возбуждено исключение, и для случая "нормального" выполнения кода (без исключений). Например, нам требуется реализовать **"rollback" **в случае возникновения исключения, то есть откат всех изменений, внесенных с определенной точки. Вот тут-то нам и помогла бы uncaught\_exception:
```
ScopeGuard::~ScopeGuard () {
    
    if( uncaught_exception() ) {
        // Было исключение, поэтому делаем Rollback (а-ля "Undo")
        Rollback();
    }
}
```
Всё замечательно и просто, за исключением того, что... этот код не будет правильно работать вот в каком случае:
```
#include <exception>

// Допустим есть класс Transaction, работающий с базой данных, 
// который в случае возникновения исключения должен откатить все изменения:
// принцип "атомарности" - "всё или ничего".

class Transaction {
public:
    ~Transaction() {
        if( std::uncaught_exception() ) 
            Rollback(); // Откат изменений
    }    
};

// Теперь представим, что у нас есть функция LogStuff, 
// которая используется повсеместно в программе: она пишет какие-то данные
// в файл, на экран и также в базу данных (использует Transaction)

void LogStuff() {
    // ... много полезной работы 
    
    Transaction t;
    // ... много полезной работы
}

// Допустим мы написали LogStuff так, что она никогда не испускает исключений,
// а значит можно спокойно вызвать её внутри деструкторов других объектов.

// Введем класс Foo, который использует LogStuff в своем деструкторе:
class Foo {
public:
    ~Foo() {
    	LogStuff(); // Хотим, чтобы лог попал в базу данных.
    }    
};


// Пример программы:
int main() {
    try {
        Foo foo;
	    throw 1;
    }
    catch(...) {
    }
}
```
Во время выполнения программы возникло исключение ( throw 1 ). Ничего страшного в этом нет, и мы готовы его обработать. Но перед этим, в процессе раскрутки стека, будут вызваны деструкторы локальных объектов, в том числе и **foo**. В деструкторе foo вызывается функция LogStuff, чтобы всю информацию внести в базу данных. Для этого создается объект класса Transaction, и выполняются все необходимые действия.  Обратите внимание, что функция LogStuff отрабатывает без ошибок, а значит данные должны попасть в базу. Но! Когда вызывается деструктор объекта Transaction (по выходу из функции LogStuff), uncaught\_exception() вернет **true**. Так как мы находимся в процессе раскрутки стека, и есть активное исключение. И все наши данные лога будут "откатаны" и потеряны. И хотя проблем с LogStuff никаких не случилось, наша транзакция среагировала на "внешнее" исключение, в то время когда, нас интересовали _только исключения с момента создания объекта Transaction и до момента его уничтожения_. Другими словами, важно не то \- было ли исключение вообще, а то \- было ли исключение в определенной области видимости. Именно этот пример приводит Саттер в своем письме комитету.

Поэтому ScopeGuard нельзя написать используя uncaught\_exception. Ведь уже название подчеркивает, что Guard "охраняет" определенный "Scope", а не распространяется на всю программу.

### **Александреску**

В настоящее время в стандартной библиотеке С\+\+ (включая стандарт С++14) нет ничего, что помогло бы отличить успешное выполнение кода в **_определенной области видимости_**, от ситуации, когда **_именно в этой области_** возникло исключение.

Однако, Андрей Александреску придумал, как реализовать ScopeGuard: он предложил ввести функцию **int getUncaughtExceptionCount()**, которая возвращала бы **сколько **исключений активно в данный момент (т.е. возбуждено, но не обработано). Если мы будем располагать такой информацией, то нам удастся реализовать настоящий ScopeGuard. Вот решение от Александреску:
```
// Класс-помощник
class UncaughtExceptionCounter {
    
	int getUncaughtExceptionCount() noexcept; // та самая функция!
	int exceptionCount_; // Счетчик, который "запомнит" кол-во исключений 
    // в момент, когда UncaughtExceptionCounter будет создан
    
public:
	UncaughtExceptionCounter()￼ 
          : exceptionCount_(getUncaughtExceptionCount()) // инициализация счетчика
    {}
    
    // Теперь можно в любой момент узнать, возникло ли новое исключение,
    // начиная от создания объекта UncaughtExceptionCounter
    bool isNewUncaughtException() noexcept {
         return getUncaughtExceptionCount() > exceptionCount_;
 	}
};


// Правильный ScopeGuard, который реагирует только на "свою" область видимости:
template <typename FunctionType, bool executeOnException>    
class ScopeGuardForNewException 
{
	FunctionType function_;
	UncaughtExceptionCounter ec_;
    
public:    
    // В конструктор передаем то, 
    // что должно "вызваться" ScopeGuard в деструкторе 
    explicit ScopeGuardForNewException(const FunctionType& fn)
        : function_(fn) {
	}
    explicit ScopeGuardForNewException(FunctionType&& fn)
		: function_(std::move(fn)) {
	}
    
	~ScopeGuardForNewException() noexcept(executeOnException) {
        
        // Здесь мы однозначно понимаем, было ли исключение 
        // в нашей области видимости и делаем соответствующие действия
        
		if (executeOnException == ec_.isNewUncaughtException()) {
			function_();
		} 
    }
};
```
Александреску указал, что реализовать функцию **int getUncaughtExceptionCount()** сегодня возможно на всех основных компиляторах С++. А Саттер, помня о схожей (но не удачной) функции uncaught\_exception, предложил переименовать getUncaughtExceptionCount() в std::uncaught\_exception**s**, просто добавив в конце "s". И вот результат:

[http://en.cppreference.com/w/cpp/error/uncaught\_exception](http://en.cppreference.com/w/cpp/error/uncaught_exception)

В С++17 появится функция std::uncaught\_exceptions, в замен "старой" std::uncaught\_exception. Эта функция позволит реализовывать классы вида ScopeGuard, которые в свою очередь позволят обрабатывать ошибки в декларативном виде, **не используя явно try/catch** во многих ситуациях. Примеры такого подхода Вы сможете найти в [N4152](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4152.pdf). В конце этого документа приложена презентация Андрея Александреску. Изучая примеры использования, начинаешь понимать, что нам может дать std::uncaught\_exceptions. Вот самый простой случай:
```
// Как-то так мы делаем сейчас
void login() {
    try {
        // полезная работа
    }
    catch (...) {
        cerr << "Failed to log in.\n";
        throw;
    }
}


// Используя ScopeGuard мы можем создать несколько полезных макросов:
// SCOPE_EXIT, SCOPE_FAILURE и SCOPE_SUCCESS и тогда обработка ошибок
// примет декларативный вид:

void login() {
    
	SCOPE_FAIL { cerr << "Failed to log in.\n"; };
    // полезная работа
}

// Можно выполнять какие-то действия, только в случае успешного 
// выполнения блока:
int string2int(const string& s) {
    
	int r;
    SCOPE_SUCCESS { assert(int2string(r) == s); };
    
    // полезная работа
	return r;
}

// Можно применять идиому RAII повсюду, без введения дополнительных типов:
void fileTransact(int fd) {
    
    enforce(flock(fd, LOCK_EX) == 0); // изначально не поддерживает RAII...    
    
    // но будет выполнено в любом случае (а-ля finally)
    SCOPE_EXIT { enforce(flock(fd, LOCK_UN) == 0); };
    
    // полезная работа    
}
```
Примеров здесь можно привести очень много, например, с вложенными блоками try/catch, которые можно переписать с использованием ScopeGuard в намного более понятной форме. И всё это, есть продолжение идиомы RAII, которую все мы очень любим. Поэтому комитет принял предложение ввести uncaught\_exceptions, взамен "старой" uncaught\_exception. Благодаря стараниям Саттера и Александреску.