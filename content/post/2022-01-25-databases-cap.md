---
title: "Согласованность, Репликация и Базы Данных по CAP"
author: ""
type: ""
date: 2022-01-25T12:00:00+03:00
subtitle: ""
image: ""
tags: []
url: /russian/2022/01/25/databases-cap
private: false
---
Это ещё одна статья про CAP теорему. Я прочитал множество книг и статей по распределенным системам и в этой статье хочу обобщить полученную информацию.
В статье я рассматриваю модели согласованности данных, типы репликации данных, свойства CAP теоремы, соотношу CAP теорему с типами баз данных.
В конце я подвожу итог в котором объясняю почему CAP теорема является формальным и условным описанием распределенных систем и почему на нее не стоит полагаться.
<!--more-->

Перед тем как перейти к разбору CAP-теоремы, рассмотрим модели согласованности данных и типы репликации данных.

## Модели согласованности данных
Модель согласованности описывает то, как видят сохранение или изменение данных разные части распределенной системы.
Например, есть некий кластер из N нод, в котором находится таблица, в этой таблице хранится переменная `A=0`.
Есть три сервиса, которые работают с этим кластером.
Допустим, первый сервис изменил переменную на `A=5`, затем второй сервис изменил ее на `A=10`, 
и после этого третий сервис считывает значение переменной `A`. 
В зависимости от используемой модели согласованности данных третий сервис может считать любое из трех значений: `0`, `5`, `10`.

Существует несколько моделей согласованности данных, если вам интересна эта тема, то обратите внимание на ссылки в конце статьи.
Тут я разберу только две обобщенные модели согласованности необходимые для понимания темы.

### Строгая (Strong consistency)
Строгая модель предполагает, что сразу же после сохранения новых данных на ноду в кластере, 
все сервисы работающие с этим кластером должны быть способны прочитать новые данные. 
Этого можно достичь двумя путями: либо сохранять новые данные сразу же на все ноды,
либо сохранять и считывать данные в режиме кворума, об этом я расскажу позже.

В примере выше, третий сервис обязательно прочитает самое последнее значение переменной равное `A=10`.

### Слабая (Weak Consistency)
Слабая модель предполагает, что после сохранения новых данных на ноду в кластере, данные могут обновляться на всех нодах кластера 
в течении какого-то периода времени. 
Некоторые ноды в кластере получат новые данные раньше, а некоторые позже, но в конечном итоге на всех нодах кластера появится эти новые данные.
Период с момента сохранения новых данных на одну ноду до появления этих данных на всех нодах кластера называется окном несогласованности (inconsistency window).

В примере выше, третий сервис может считать любое из трех значений переменной `A`: `0`, `5`, `10`.
Значение будет зависеть от того к какой ноде обратился сервис и сколько прошло времени с момента последнего обновления данных на этой ноде.

## Типы репликации данных
Репликация - это процесс синхронизации данных между несколькими нодами распределенной системы в кластере.

Например, есть три ноды базы данных и на одну из нод кто-то записывает новые данные. 
С помощью механизма репликации новые данные с этой ноды будут синхронизированы с двумя другими нодами.
В результате на всех трех нодах будут одинаковые данные.

Репликация может потребоваться для:
1. Бекапа, если что то случится с одной из нод, то останется еще несколько копии данных.
2. Отказоустойчивости, то же самое что и в пункте 1, если одна из нод выйдет из стоя, то останется еще несколько нод.
3. Распределения нагрузки, при большом количестве запросов, эти запросы можно равномерно распределить между несколькими нодами.

Я приведу два основных типа репликации данных. Иногда встречаются вариации этих двух типов в зависимости от возможностей конкретной базы данных.

### master-slave
При `master-slave` репликации, есть одна главная нода - `master`, которая обрабатывает запросы на запись и чтение данных. 
К главной ноде подключены еще `N` нод - `slave`, которые являются полными копиями `master` ноды. 
В большинстве конфигураций на `slave`-ноду нельзя записать данные, эти ноды обрабатывают только запросы на чтение.
После добавления новых данных `master`-ноду данные реплицируются на все `slave`-ноды.

Если `master`-нода выходит из строя, то выбирается новая `master`-нода из `slave`-нод. 
После чего, все запросы на запись идут в новую `master`-ноду. 
Иногда процесс выбора новой главной ноды выполняется вручную, через изменение конфигурации, 
а иногда автоматически. Временами имеется возможность восстановить вышедшую из строя `master`-ноду.

Во время смены `master`-ноды существует риск потерять данные. 
Например, клиент что-то записал на `master`-ноду, после чего `master`-нода упала и данные не успели реплицироваться на `slave`-ноды.
Теперь после выбора новой `master`-ноды из живых `slave`-нод, те данные потеряются.

### multi-master (одноранговая, master-master, P2P)
При `multi-master` репликации все ноды являются `master`-нодами и все они главные.
Каждая нода умеет обрабатывать запросы на запись и на чтение. 
После записи данных на одну `master`-ноду, данные реплицируются на остальные `master`-ноды.

Если выйдет из стоя одна `master`-нода, то оставшиеся `master`-ноды продолжат обрабатывать запросы.

### Синхронная и Асинхронная репликация данных
Данные между нодами могут реплицироваться двумя способами - синхронно и асинхронно.

Синхронная репликация выглядит так:
1. Клиент устанавливает соединение с `master`-нодой и присылает ей новые данные.
2. `master`-нода сохраняет новые данные к себе и синхронно отправляет эти данные в другие ноды.
3. Все остальные ноды сохраняют новые данные к себе и сообщают `master`-ноде, что сохранение прошло успешно.
4. `master`-нода, сообщает клиенту что сохранение новых данных прошло успешно и разрывает с ним соединение.

В случае синхронной репликации, запрос на сохранение данных для клиента занимает достаточно долгое время. 
Клиент должен дождаться, когда все ноды сохранят данные, только после этого клиент получит ответ об успешном сохранении.

Асинхронная репликация выглядит так:
1. Клиент устанавливает соединение с `master`-нодой и присылает ей новые данные.
2. `master`-нода сохраняет данные к себе, затем сообщает клиенту, что сохранение данных прошло успешно и разрывает соединение с клиентом.
3. Затем в фоне все остальные ноды забирают у `master`-ноды данные и сохраняют их к себе.

В случае асинхронной репликации, запрос на сохранение данных для клиента занимает меньше времени,
потому что клиент не ждет сохранения данных на все ноды. 
Но за повышение скорости сохранения взимается плата в виде ослабления модели согласованности данных и увеличения риска потери данных.

### Кворум
Кворум используется при репликации `multi-master`, когда:
* необходимо повысить надежность сохранения данных
* необходимо получить строгую модель согласованности данных

Кворум - это режим записи данных сразу на несколько нод и чтения данных, сразу с нескольких нод.

Количество нод входящих в кворум считается по формуле `(n/2 + 1)`. 
Например, при `5`-ти нодах запись и чтение будет выполняться на `3` ноды сразу.

Кворум позволяет повысить надежность записи данных, потому что данные будут записаны сразу на несколько нод, 
а вероятность выхода из строя сразу нескольких нод минимальна.
Но иногда, из строя может выйти целый дата-центр. 
Если требуется очень большая устойчивость к сбоям, но разные ноды одной БД располагают сразу в нескольких дата центрах.
В случае наличия нескольких дата центов, данные сохраняются сразу в несколько `master`-нод в разных дата центрах.

При чтение данных в режиме кворума запрос на чтение отправляется сразу на несколько нод,
результаты чтения сортируются по версиям и данные с самой последней версией отдаются клиенту.
Это гарантирует, что клиент всегда получит самую последнюю версию данных и позволяет добиться строгой модели согласованности данных.

Есть еще более гибкий подход - задавать количество нод, на которое требуется записать данные и с которых необходимо считать данные.
Что бы достичь строгой согласованности, следует придерживаться формулы `W + R > N`, 
где `W` - количество нод на которые выполняется запись,
`R` - количество нод с которых выполняется чтение,
`N` - количество реплик с данными (replication factor).
Достаточно надежным считается количество реплик `N=3` и выше.

Например, есть три реплики `N=3`, у сервиса мало запросов на запись, но много запросов на чтение. 
Тогда для достижения строгой согласованности эффективнее писать на `W=3` ноды, а считывать с `R=1` нод. 
Условие `3 + 1 > 3` выполняется, сервис получает строгую согласованность данных и требуемую скорость чтения.

## CAP
CAP теорема описывает компромиссы при разработке и использовании распределенных систем. 
Она применяется для базы данных, либо для распределенных файловых системы или хранилищ объектов вроде S3, HDFS.

Три буквы в слове CAP, являются свойствами распределенной системы: Consistency, Availability, Partition tolerance.
Теорема гласит, что возможно выбрать только два из трех свойств. Следовательно, необходимо пожертвовать одним из свойств.
Каким свойством жертвовать, зависит от требований к конкретной распределенной системе.

Разберем, что означает каждое свойство из CAP теоремы.

### Consistency (Согласованность)
Под согласованностью в CAP теореме понимается строгая модель согласованности данных. 
Следовательно, клиент всегда должен видеть самую последнюю и свежую версию данных.

Как я уже писал выше, строгой модели согласованности можно добиться двумя путями:
1. Использовать синхронную репликацию.
2. Использовать асинхронную репликацию с записью и чтением данных в режиме кворума.

### Availability (Доступность)
Под доступностью в CAP теореме понимается, что если нода в рабочем состоянии, то она должна успешно отвечать на любой запрос.

Свойство доступности в CAP не учитывается лейтенси ([latency](https://en.wikipedia.org/wiki/Latency_(engineering))),
что является одним из недостатков CAP.
Например, если нода отвечала на запрос 10 минут, то с точки зрения доступности в CAP это нормально и нода считается доступной.
В реальных системах такое большое лейтенси является проблемой и нода считается недоступной.

Поэтому следует сделать допущение и считать, что нода считается доступной, если она отвечает на запрос за разумный период времени, например за 200мс.
Это допущение противоречит оригинальной CAP теореме, но большинство делает его, потому что оно ближе к действительности.

### Partition tolerance (Устойчивость к разделению)
Под устойчивостью к разделению в CAP теореме понимаются проблемы с сетью.
Это могут быть задержки, потери пакетов или полный разрыв соединения ([split-brain](https://en.wikipedia.org/wiki/Split-brain_(computing)))
в результате которого ноды не видят друг друга.

Распределенная система, состоящая из нескольких нод подразумевает, что она имеет дело с сетью.
Следовательно, нельзя пожертвовать этим свойством CAP теоремы, 
неустойчивая к сетевым проблемам распределенная система работает некорректно.

## Комбинации CAP свойств
CAP теорема гласит, что возможно выбрать только два из трех выше описанных свойств.
В результате получаются три комбинации: CP, AP, CA.
Разберем основные свойства каждой из этих комбинаций.

### CP: Выбираем Consistency и Partition tolerance, жертвуем Availability
Распределенная система устойчива к сетевым проблемам и соответствует строгой модели согласованности,
но при проблемах с сетью часть системы становится недоступной.

Например, при `master-slave` репликации одна из `slave`-нод может стать недоступной для чтения,
но `master`-нода продолжит принимать запросы на запись и чтение.
А вот если `master`-нода выйдет из строя, то БД будет недоступна для записи, но `slave`-ноды продолжат обрабатывать запросы на чтение.

Еще можно привести пример с двумя дата центрами. 
Допустим есть два дата центра в разных регионах страны. Если в какой то момент соединение между ними теряется,
то второй дата центр должен перестать принимать запросы на чтение и запись,
а приложения работавшие со вторым дата центром, должны переключиться на первый рабочий дата центр.

### AP: Выбираем Availability и Partition tolerance, жертвуем Consistency
Распределенная система устойчива к сетевым проблемам и всегда доступна, но использует слабую модель согласованности. 
При сетевых проблемах разные ноды будут выдавать разные данные на один и тот же запрос из-за отсутствия репликации между ними.

Например, при `multi-master` конфигурации во время потери связи между нодами (split-brain) репликация остановится,
но ноды продолжат работать независимо, хоть и не будут знать об новых данных друг у друга.

Или например, при `master-slave` конфигурации во время потери связанности с `master`-нодой `slave`-ноды продолжат выполнять запросы на чтение, 
но они будут отдавать только старые данные.

В примере с двумя дата центрами, когда между ними происходит разрыв соединения, оба дата центра становятся на время независимыми и 
не видят обновления друг у друга, в результате чего часть данных в них устаревает до момента возобновления соединения.

### CA: Выбираем Consistency и Availability, жертвуем Partition tolerance
Как я уже писал выше в распределенной системе нельзя пожертвовать `Partition tolerance`, потому что распределенная система всегда имеет дело с сетью.
К этой категории возможно "с натяжкой" отнести базы данных работающие в режиме одной нодой на одном сервере без репликации данных.

## Типы баз данных
Рассмотрим популярные типы баз данных с точки зрения CAP теоремы.

### Реляционные базы данных (Relational Database, RDBM)
Реляционные БД один из самых старших и зрелых видов БД, они являются одними их самых гибких в настройке.
Большинство из популярных баз поддерживают как `master-slave`, так и `multi-master` репликацию, синхронную и асинхронную.
Базу возможно сконфигурировать как со строгой согласованностью данных так и со слабой.

Реляционные БД могут попадать в категорию CP, либо AP по CAP в зависимости от типа репликации данных и настроек.

### Документные базы данных (Document-oriented database)
Типичный представитель этого типа баз данных - MongoDB.
Она поддерживает только асинхронную `master-slave` репликацию данных.
В конфигурации по умолчанию чтение и запись выполняется через `master`-ноду, БД поддерживает строгую согласованность данных и является CP по CAP.

При отказе `master`-ноды, будет автоматически выбрана новая `master`-нода, но на новой ноде может не оказаться новых данных из-за асинхронной репликации.
Поэтому при автоматическом выборе новой `master`-ноды MongoDB гарантирует только слабую согласованность.

В MongoDB возможно сконфигурировать чтение с `slave`-нод и уровень согласованности данных при репликации между `master` и `slave`.
Благодаря этим настройкам возможно добиться нужного уровня согласованности данных при чтении с `slave`-нод и достичь либо CP, либо AP по CAP.

Делаем вывод, что в зависимости от конфигурации MongoDB следует рассматривать как CP, либо как AP по CAP.

### Column-family databases
Прародитель этого типа баз данных - [Google BigTable](https://static.googleusercontent.com/media/research.google.com/ru//archive/bigtable-osdi06.pdf).

Наиболее популярным open source представителем этой категории баз данных является Cassandra, о ней и поговорим.
Cassandra с самого начала создавалась как распределенная база данных для высоконагруженных систем с упором на масштабируемость.

Эта БД поддерживает только асинхронную `multi-master` репликацию данных. 
Для каждого запроса требуется задать [уровень согласованности](https://docs.datastax.com/en/cassandra-oss/3.0/cassandra/dml/dmlConfigConsistency.html).
Если указать значение меньшее чем количество нод в кворуме, то получится слабая согласованность данных,
а если указать кворум, то получится строгая согласованность данных.

Из этого можно сделать вывод, что в зависимости от запроса эта БД может относится как к CP, так и к AP типу по CAP теореме.
Благодаря возможности задать уровень согласованности прямо в запросе, некоторые запросы могут быть CP типа, а некоторые AP типа.

## Компромиссы CAP теоремы
Большинство баз данных можно настроить как БД CP типа, так и AP типа по CAP. 
Базы предоставляют гибкую конфигурацию модели согласованности данных,
поэтому сложно отнести какую либо БД к одному конкретному типу по CAP.

CAP теорема обладает большим количеством недостатков, например:
* в `Availability` ничего не сказано про лейтенси ([latency](https://en.wikipedia.org/wiki/Latency_(engineering)))
* под `Consistency` понимается только строгая модель согласованности данных
* CAP теорема не учитывает транзакции

Подробнее про недостатки CAP теоремы можно почитать в статье [Please stop calling databases CP or AP](https://martin.kleppmann.com/2015/05/11/please-stop-calling-databases-cp-or-ap.html) ([Перевод на русский](https://habr.com/ru/post/258145/)).

Не стоит считать CAP теорему истиной в последней инстанции.
К CAP следует относится как к формальной теореме для приблизительного понимания компромиссов существующих в распределенных системах.
Из-за большого количества недостатков распределение БД по CAP очень условное, т.к.
БД имеют гибкие настройки согласованности данных, которые сложно учесть при соотнесении их с CAP теоремой.

CAP теорему следует использовать только при поверхностном описании распределенной системы, 
что бы слушатель/читатель мог примерно понять каких свойств по доступности и согласованности данных вы пытаетесь достичь.

Более точным подходом будет выбор между строгостью модели согласованности данных и получаемым в результате лейтенси. 
Чем выше согласованность данных, тем больше времени нужно на выполнение запросов и следовательно тем выше будет лейтенси. 
И наоборот, при слабой согласованности запросы выполняются быстрее и лейтенси уменьшается.
Смещение в ту или иную сторону зависит конкретной предметной области и требований к системе.

## Дальнейшее чтение
* [NoSQL Distilled Book](https://www.amazon.com/gp/product/0321826620)
* [Overview of Consistency Levels in Database Systems](http://dbmsmusings.blogspot.com/2019/07/overview-of-consistency-levels-in.html)
* [Please stop calling databases CP or AP](https://martin.kleppmann.com/2015/05/11/please-stop-calling-databases-cp-or-ap.html) ([Перевод на русский](https://habr.com/ru/post/258145/))
* [Consistency model Wikipedia](https://en.wikipedia.org/wiki/Consistency_model)
* [Big Data World, Part 5: CAP Theorem](https://blog.jetbrains.com/blog/2021/06/03/big-data-world-part-5-cap-theorem/)
* [Мифы о CAP теореме](https://habr.com/ru/post/322276/)
