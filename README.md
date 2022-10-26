# Решение домашнего задания №1
Выполнение данной работы будем производить на платформе 
[Яндекс.Облако](https://cloud.yandex.ru/)

## Создание виртуальной машины

Регистриуемся на платформе Яндекс.Облако

Переходим в личный кабинет

Создаем виртуальную машину с названием __postgre__ на *Ubuntu 22.04 LTS*, используя стандартные настройки Compute Cloud
![Параметры созданной ВМ](imgs/0.png)

Во время создания виртуальной машины настраиваем подключение к ВМ пользователю по ssh, используя публичный ssh-ключ, сегенерированный с помощью команды 
 > ssh-keygen -t rsa

в командной строке

![Создание ключа SSH](imgs/1.png)

 > В моем случае, так как на ПК установлена ОС Windows 11, в качестве командной строки Linux я буду использовать инструментарий Windows для Linux (_wsl_ с установленной через PowerShell Ubuntu 22.04)

После создания и включения виртуальной машины, подключаемся к ней, используя _cli_

 Выполняем подключение к виртуальной машине, используя ее внешний ip-адрес

 ![Подключение к ВМ](imgs/2.png)

## Настройка виртуальной машины

 Чтобы иметь возможность подключаться вторым окном, необходимо создать еще одного пользователя _testuser_, добавить его в группу _sudo_, сгенерировать ему ключ ssh и прописать его в файл
 > .ssh/authorization_keys

 После создания и настройки второго пользователя, осуществим установку PostgreSQL 15 версии, используя команду
 > sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15

 ![Установка PostgreSQL](imgs/3.png)

 После установки PostgreSQL автоматически настроит кластер с названием __main__ на порту по умолчанию __5432__

 Просмотреть все созданные кластеры можно используя команду 
 > pg_lsclusters

 ![Просмотр кластеров PostgreSQL](imgs/4.png)

 ## Настройка базы данных для дальнейших работ

Войдем в консоль PostgreSQL, используя учетную запись postgres и порт по умолчанию, в обеих консолях
 ![Вход в консоль PostgreSQL](imgs/6.png)

> __Примечание.__ На одной машине могут быть развернуты несколько класетров Postgre, они будут отличаться портом и директорией расположения данных. Для подключения к конкретному кластеру необходимо добавить к команде ключ __-p %portNumber%__, где _%portNumber%_ - порт необходимого кластера)

Создадим базу данных _learnpsql_

![Создание БД](imgs/7.png)
 
 и отключим автокоммит

 ![Отключение автокоммита](imgs/8.png)

 Выполним подключение к созданной нами ранее БД _learnsql_, используя команду
 > \c learnsql

 ![Подключение к БД](imgs/9.png)

 Создадим необходимую таблицу с полями и заполним ее тестовыми данными
 > Так как у нас установлен режим __AUTOCOMMIT OFF__, то данную транзакцию необходимо завершить вручную, используя команду __COMMIT__

 ![Создание и заполнение таблицы](imgs/10.png)

 Проверим во втором окне, что после COMMIT мы видим данную таблицу с данными

 ![Проверка таблицы](imgs/11.png)

 ## Выполнение операций при разных уровнях изоляции транзакций

 Проверим текущий уровень изоляции транзакций, выполнив команду
 >  show transaction isolation level;

 ![Проверка текущего уровня изоляции транзакций](imgs/12.png)

 По умолчанию PostgreSQL устанавливает уровень изоляции транзакций __READ COMMITTED__.
 > Уровень изоляции транзакций __READ COMMITTED__ означает, что внутри транзакции мы увидим лишь те данные, которые были изменены/добавлены в подтвержденных параллельно выполняющихся транзакциях (завершившихся успешным COMMIT) либо те данные, которые мы изменили/добавили в текущей транзакции.

 ### Уровень изоляции транзакций READ COMMITTED

 Откроем транзакции в обоих подключениях, в первом окне добавим новую строку в таблицу _Persons_ и выберем строки, во втором просто выберем строки

 ![Проверка уровня изоляции транзакций READ COMMITTED до COMMIT](imgs/13.png)

 Как видим, во втором окне новую строку не видно, так как уровень изоляции транзакций установлен в READ COMMITTED, а это значит, что новая строка существует лишь в той транзакции, в которой она была добавлена в таблицу, до совершения COMMIT.

Осуществим COMMIT в первом окне и проверим, появилась ли строка для подключения во втором окне

![Проверка уровня изоляции транзакций READ COMMITTED после COMMIT](imgs/14.png)

Как видим, после успешного COMMIT данная строка появилась для всех остальных пользователей. Это обусловлено тем, что запрос в транзакции данного уровня видит снимок данных на момент начала текущего оператора в транзакции (не считая команд управления транзакциями). На момент выполнения SELECT транзакция из первого окна была зафиксирована COMMIT.

 ### Уровень изоляции транзакций REPEATABLE READ

Переключим уровень изоляции транзакций в режим __REPEATABLE READ__
> Уровень изоляции транзакций __REPEATABLE READ__ означает, что внутри транзакции видимы лишь те данные, которые были зафиксированы на _начало транзакции_, а не на начало текущего оператора (в отличии от __READ COMMITTED__, где данные могут быть изменены параллельной транзакцией в ходе выполнения текущей транзикции)

![Переключение уровня изоляции транзакций в режим REPEATABLE READ](imgs/15.png)

Откроем транзакции в обоих окнах, в первом окне добавим новую строку в таблицу, во втором проверим, появилась она или нет

![Проверка уровня изоляции транзакций REPEATABLE READ до COMMIT](imgs/16.png)

Как видно, во втором окне новой строки не существует в таблице с данными, так как уровень изоляции транзакций __REPEATABLE READ__ подразумевает видимость только тех данных, которые были зафиксированы до начала транзакции, но не допускает видимость незафиксированных данных и изменений, произведённых другими транзакциями в процессе выполнения данной транзакции.

Осуществим COMMIT в первом окне и проверим, появились ли данные во втором окне в той же транзакции

![Проверка уровня изоляции транзакций REPEATABLE READ после COMMIT в первом окне](imgs/17.png)

Как видим, данные не появились во втором окне. Это обусловлено тем, что запрос в транзакции данного уровня видит снимок данных на момент _начала первого оператора в транзакции_ (не считая команд управления транзакциями), а не начала текущего оператора. 

Выполним COMMIT во втором окне и снова выберем данные из таблицы

![Проверка уровня изоляции транзакций REPEATABLE READ после COMMIT во втором окне](imgs/18.png)

Как видим, данные появились, так как была во втором окне была завершена транзакция, для которой новой строки не существовало.

