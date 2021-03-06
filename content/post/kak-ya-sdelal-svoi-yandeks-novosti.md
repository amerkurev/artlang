---
author: "Andrey Merkurev"
linktitle: "Как я сделал свои «Яндекс.Новости»"
title: "Как я сделал свои «Яндекс.Новости»"
publishDate: 2017-10-27T00:00:00+03:00
date: 2017-10-27T00:00:00+03:00
draft: false
tag: "python, django, fasttext, reactjs, news aggregator, rss, yandex"
---

История создания агрегатора на базе технологий искусственного интеллекта<br>«[Новостной робот](https://newsbot.press/)».

### **Вместо вступления**

В этой статье речь пойдёт о новостном агрегаторе Newsbot.press. Почему этот агрегатор я сравниваю именно с «Яндекс.Новостями»? Потому что он полностью автономный, все решения принимает сам. Внутри — машинное обучение, нейронные сети. Он намного сложнее традиционных агрегаторов. Но обо всём по порядку. Начну с того, кто разработал эту систему и зачем.

Я программист, пишу на C++ и Python, знаю ещё пару языков. Работаю в небольшой фирме в Москве. Это далеко не «Яндекс» и не Google. Но я очень люблю своё дело. По этой причине в свободное время я изучаю фреймворки, пробую новые сервисы, пишу свои пет-проекты.

Именно из такого пет-проекта и вырос «Новостной робот», агрегатор с ИИ. Весь проект от фронтенда до бэкенда, от микросервисов до алгоритмов машинного обучения, от работы с облаками до интеграции в соцсети — всё это написал один человек. Это не продукт компании или команды людей. И я не рекламирую и не пытаюсь продать вам свой продукт. Я хочу рассказать интересную историю.

### **Новостной робот**

Новостной робот — это автоматизированная система сбора и анализа новостной информации. Задача робота: собрать информацию с сайтов СМИ, выделить главные новости в разных категориях и сгруппировать их по темам.

Новостной робот никогда не публикует фото или видеоматериалы, не публикует инфографику и исследования, авторские статьи и мнения. Я попытался исключить любое возможное нарушение авторских прав. Поэтому робот работает только с сообщениями о новостях дня.

Чтобы сайты СМИ получали прибыль от показа рекламы, полные тексты новостей тоже никогда не публикуются. Описание новости всегда ограничено одним или двумя предложениями. Название источника и прямая ссылка на новость в источнике всегда присутствуют.

Новостной робот не создаёт новую информацию. Все заголовки и цитаты представлены в том виде, в котором они были получены из источников.

Робот полностью автономен. Ни я, ни кто-либо ещё не принимает участия в его повседневной работе. Он всё делает самостоятельно, в том числе выбирает главные темы дня или срочные новости последних часов. Аналогично работают и «Яндекс.Новости». Хотя их часто и винят в том, что «ручное» управление всё-таки имеет место. Но я в это слабо верю.

Внешне сервис достаточно прост, вряд ли стоит описывать что-то ещё. Лучше перейти к деталям реализации и мотивам, побудившим меня на создание «Новостного робота».

### **Проблема в рекламе**

Помните 19 сезон «Южного парка»? Реклама проникает во все сферы жизни и начинает манипулировать людьми. Надеюсь, мы далеки от такого. Но уже сейчас интернет полон рекламы, кажется, что её больше, чем на телевидении и в газетах.

В дело вступают компании-гиганты: Google и Facebook. Компании, которые сами зарабатывают на рекламе. От этого рождаются очень странные решения. [Процитирую](https://t.me/addmeto) Григория Бакунова: «Рубрика пчёлы против мёда: в текущем обновлении Chrome для Android появился встроенный блокировщик рекламы. Это выигрышная стратегия компании — блокировщик рекламы будет блокировать всё, кроме рекламы от Google. Все проиграли, Google выиграла».

Facebook пытается предложить модель партнерства со СМИ, но что из этого выйдет, пока неясно. Зато война Facebook с AdBlock с переменным успехом идёт уже давно.

До того, как я стал пользователем «Новостного робота», меня в новостях не устраивало следующее. Я люблю быть в курсе происходящего, но большинство новостей, особенно политических, мне не интересны.

Мой обычный кейс: захожу на свой любимый сайт СМИ (зачастую со смартфона), сразу появляется баннер, который закрывает мне весь экран телефона. Убираю баннер. Под ним уже ждёт вторая реклама, её можно просто пролистать. Иногда без предупреждения страница разъезжается, и в центре появляется рекламное видео, которое сразу начинает играть.

Когда я наконец-то вижу список новостей, оказывается, что там для меня нет ничего интересного и читать дальше я не буду. Получается, я потратил трафик, посмотрел рекламу, а пользы для себя не извлёк. И этот кейс повторяется несколько раз за день. Поэтому я и создал «Новостного робота».

Я готов посмотреть любое количество рекламы в материале, который мне хочется прочитать подробно. Но это должен быть осознанный выбор. И выбирать я хочу без рекламы. Только когда я выбрал тему и источник, я перехожу на сайт, где читаю статью подробнее уже с рекламой. Особенно если речь идёт о событиях дня.

На сайте «Новостного робота» реклама полностью отсутствует. И так будет всегда. Это для меня теперь основная точка входа, когда речь идёт о новостях.

### **Почему не AdBlock**

Есть те, кто качает фильмы и музыку только с торрентов. Те, кто использует софт в коммерческих целях и не платит за него. И много тех, кто ставит себе AdBlock в браузере. И не только себе. «Продвинутый гик» непременно установит AdBlock и своей маме, и всем её знакомым. И те будут ему благодарны. Но я уверен, в глубине души вы понимаете, что это воровство. Хотя конечно, есть ужасные сайты, на которые лучше не попадать.

### **Детали реализации**

Теперь о «Новостном роботе» с технической стороны. Вдруг это будет кому-то полезно. Всю систему можно разбить на три большие части: это краулер, семантический анализатор и сам сайт.

Краулер занимается сбором информации. И тут есть интересный момент. Например, «Яндекс» собирает информацию с сайтов с помощью специальных регламентируемых RSS-потоков. При этом ресурс-источник должен предоставить «Яндексу» несколько потоков: один — для «Новостей», другой — для «Дзена». А вот для Google News, кажется, не нужно ничего делать, главное, чтобы новостной сайт индексировался Google.

Я не имел доступа к RSS-потокам «Яндекса». Поэтому пошёл по пути Google. Создать универсальный эвристический алгоритм мне не удалось. Тогда я просто научил краулера забирать данные и из RSS, и с сайта. При этом я практически вручную подключаю источники. Чтобы делать это в пару кликов, мне пришлось написать отдельный сайт для краулера, подобие админки.

Краулер не нагружает сайты, на которые ходит. Так как делает он это в три раза реже, чем, к примеру, робот «Яндекса». Он берёт не всё, а только новые данные. Краулер написан на Python. Веб-сайт Краулера сделан на Django. Этот сайт намного сложнее, чем основной, который практически полностью состоит из генерируемой статики.

На фронтенде везде используется React от Facebook, состоянием управляет Redux. Я мог бы всё это сделать и на Angular. Вот только первый уже не актуален, а ко второму нужен TypeScript. Его я пока не знаю. Вообще я не большой специалист по фронтенду. Зато очень недурно готовлю Django.

«Новостной робот» использует облака: несколько виртуальных серверов, облачное хранилище и прочее. С самого начала я решил, что робот должен быть максимально экономичен в плане ресурсов. Ведь содержать его придётся мне.

Пришлось переписать некоторые библиотеки, отказаться от многих удобных облачных услуг. В итоге вся инфраструктура обходится в пару тысяч рублей в месяц, что для меня не составляет труда оплачивать самому. Но жаль, что мой бюджет был ограничен. Из-за этого программировать пришлось больше.

### **Что общего у «Новостного робота» и «Палеха»**

Семантический анализатор Новостного робота использует библиотеку [fastText](https://github.com/facebookresearch/fastText), которую разрабатывает Facebook. В одно прекрасное утро Facebook просто опубликовала в открытый доступ построенный массив векторов fastText для двухсот языков!

Снова процитирую Григория Бакунова: «По моим оценкам, на тренировку одного языка вроде русского с 300 измерениями у меня уходило две недели машинного времени. Это очень щедрый подарок для всех, кто работает с сущностями языка и строит сервисы, понимающие людей».

Я понял, что эта штука может мне сильно помочь в работе. А то, что fastText ещё и идёт вместе с готовой моделью, даёт возможность сразу приступить к экспериментам. Я не эксперт в машинном обучении и точно им не стану. Но тренд очевиден всем.

Здорово, что компании-лидеры в этой области публикуют исходники готовых решений: TensorFlow от Google, fastText от Facebook, CatBoost от «Яндекса». За это нужно сказать «спасибо», а потом брать и пробовать. И не важно, занимались ли вы до этого искусственным интеллектом и машинным обучением или нет. Неизвестно, к чему всё придёт в итоге. Быть может, «развесистые» фреймворки скроют от нас всю магию, а мы просто будем пользоваться всей мощью машинного интеллекта.

FastText позволил мне строить семантические векторы слов и даже целых фраз, а затем сравнивать их и делать выводы о том, насколько похожи сами слова между собой. «Новостной робот» стал понимать, к примеру, что «импичмент» и «отставка» близкие в определённом контексте слова.

Совсем недавно «Яндекс» объявил о запуске нового поискового алгоритма «Палех», который позволит точнее понимать запросы людей. Цель «Яндекса» — получить на основе нейронных сетей модели, способные «понимать» семантическое соответствие запросов и документов на уровне, сравнимом с уровнем человека. Для этого они развивают применение семантических векторов.

Посмотрите на то, как [работает](https://newsbot.press/) «Новостной робот», который использует семантические векторы самым прямолинейным образом. А теперь представьте, что сможет сделать «Яндекс» с его сотнями специалистов и исследователей в этой области.

Работают ли «Яндекс.Новости» с семантическими векторами, мне точно неизвестно. Я также ничего не знаю и об их алгоритмах кластеризации. Новостной робот использует для кластеризации алгоритмы из библиотеки [scikit-learn](http://scikit-learn.org/stable/).

### **Что будет дальше**

«Новостной робот» был создан за пять месяцев. Я получил огромный опыт. Сейчас система работает самостоятельно, без моего участия. Лишь иногда я исправляю ошибки, которые нахожу. Робот очень экономичен, и я могу сразу оплатить все расходы на его инфраструктуру в течение 10 лет. Никакие инвестиции мне для этого не нужны, равно как и заработок от рекламы на сайте.

Кроме меня сайтом пользуются мои знакомые и друзья. Насколько я знаю, они рассказывают о нём другим людям. «Новостной робот» хорошо подходит людям старшего поколения, которым сложно «бороться» с баннерами на телефонах.

У меня есть ещё много идей по развитию сервиса. К примеру, я обдумывал что-то вроде серверного AdBlock. «Новостной робот» мог бы рендерить сайты СМИ на сервере и понимать, где рекламы больше. А затем оценивать каждый источник в соответствии с этим показателем. Тогда пользователь мог бы выбирать источник исходя из того, где меньше рекламы. Если ему не принципиально, на каком сайте знакомиться с темой.

Есть задачи и более простые, например, интерфейсные. Я знаю, что некоторым хотелось бы иметь возможность применить фильтр на сайте и вовсе скрыть некоторые СМИ из выдачи.

«Новостной робот» слабо связан с морфологией русского языка. Мне потребуется немного времени, чтобы запустить робота для другого языка.

Однако на данном этапе я пока остановлюсь. Направлений движения слишком много, чтобы быстро выбрать правильное. Возможно, и не стоит больше ничего делать. Если аудитория робота останется прежней, то и сам робот останется таким, как сейчас. Какой бы ни была аудитория, робот продолжит свою работу и будет свободным от рекламы.

### **Про собственный опыт**

В заключение хотел бы рассказать о [Telegram-канале](https://t.me/newsbotpress) «Новостного робота». Для меня такой способ информирования о событиях дня оказался самым удобным. Я отключил уведомления в канале, чтобы они не мешали подписчикам. В день приходит около 20 коротких сообщений, каждое из которых посвящено какой-то важной теме последних часов.

В свободное время, например, в метро я открываю канал и за 15 секунд узнаю о событиях этого дня. Перехожу ли я после этого на сайт, чтобы почитать? Редко, только в тех случаях, когда тема мне интересна.