# Проект

## Тема «Анализ влияния различных файловых систем на производительность кластеров PostgreSQL»

### О проекте
Рассмотрим файловые системы EXT4, XFS, ZFS и проведем анализ Postgresql при помощи утилиты pgbench с различными настройками в postgresql.conf на разных объемах данных

Основные цели эксперимента:

  1. Проанализировать как ведет себя Postresql при изменении настроек на разных файловых системах и разных объемах данных;
  2. Определить зависимость TPS и Latency и выявить наиболее оптимальную систему для Postgresql

### Подготовительная работа
1. Разворачиваем ВМ
2. Создаем каталоги под XFS и ZFS системы (\xfs ; \zfs)

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/be528421-7c19-4bfa-8d6b-5da0a1c62f37)

3. Устанавливаем Postgresql 15
>sudo apt update && sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo DEBIAN_FRONTEND=noninteractive apt -y install postgresql-15
>
Проверяем что кластер запущен через sudo -u postgres pg_lsclusters
![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/b0df0a9f-2503-4bb4-861d-bd4dfe5b580d)

Тестировать будем на двух объемах данных с параметром множителя s=10 и s=100 соответственно
Создаем 2 БД для теста 
Для s = 10
>CREATE DATABASE postgres10;
>
Для s = 100
>CREATE DATABASE postgres100;
>

Создание таблиц для тестов:
>sudo -u postgres pgbench -i -s 10 postgres10
>
>sudo -u postgres pgbench -i -s 100 postgres100
>

Заранее скопируем data_directory в каталоги с XFS и ZFS системой.

>sudo -u postgres psql
>

Посмотрим текущий каталог:

>SHOW data_directory;
>

По умолчанию стоит каталог /var/lib/postgresql/15/main
нужно остановить PostgreSQL

>sudo systemctl stop postgresql
>
убедиться, что служба действительно остановлена
>sudo systemctl status postgresql
>

В каталог с xfs системой:
>sudo rsync -av /var/lib/postgresql /xfs
>
В каталог с zfs системой:
>sudo rsync -av /var/lib/postgresql /zfs
>

По умолчанию значение для data_directory выставлено на /var/lib/postgresql/15/main в файле /etc/postgresql/15/main/postgresql.conf. 
Его нужно отредактировать, чтобы указать новый каталог:

>sudo nano /etc/postgresql/15/main/postgresql.conf
>

Теперь в строке, начинающейся с data_directory нужно изменить путь, чтобы указать новое местоположение. Обновлённая директива будет выглядеть примерно так:
/etc/postgresql/15/main/postgresql.conf

. . .

data_directory = '/xfs/postgresql/15/main'

. . .

data_directory = '/zfs/postgresql/15/main'

Включаем
>sudo systemctl start postgresql
>
Проверяем
>sudo systemctl status postgresql
>
Убедимся что используем новый каталог
>sudo -u postgres psql
>
Ещё раз проверяем значение каталога данных:
>SHOW data_directory;
>
Аналогично будем переключаться между файловыми системами в наших тестах.

## Эксперимент №1
Используем настройки из коробки.
Изначально postgresql установлен в файловую систему EXT4
Команда для нагрузки s=10
>sudo -u postgres pgbench -c50 -P 15  -T 60 -U postgres postgres10
>

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/d04282ce-efbd-4a68-9a0b-666b93202a6f)

Команда для нагрузки s=100
>sudo -u postgres pgbench -c50 -P 15  -T 60 -U postgres postgres100
>

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/482286db-dbb5-4865-b524-5ea8e7895ca5)

Аналогично для XFS и ZFS систем:
![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/962e3e4e-b36a-407f-bf61-4a0e74fecc66)

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/a0163d36-d3fc-4cca-9e85-7189944b541f)

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/63939870-1096-4927-93a8-e3e1d67913ae)

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/b1e747c5-8d2e-4a76-ba43-e5178e4cf53f)

Получаем следующую таблицу результатов:

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/a8b669c4-9ada-4a62-a258-24439d802779)

Видим, что EXT4 и XFS существенно не отличаются, в то время как ZFS при увеличении объема резко теряет TPS  и увеличивает Latency

## Эксперимент №2
>log_min_duration_statement = -1
>
Отключим запись в журнал продолжительность выполнения всех команд, время работы которых не меньше указанного.

Проведем тестирование, аналогично эксперименту №1.
EXT4

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/986657ee-45e2-4a4a-b5df-1116b9ec3262)

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/4363a049-148e-4cfc-9e01-333acfdfb78c)

XFS

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/431b6fee-6252-4ba6-8244-de6f735ab3a4)

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/509c452d-860b-458e-9262-def3953ba508)

ZFS

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/d4c7cc69-c09c-40c2-83ec-5f78dd0c338f)

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/428ffbf0-b410-461c-9dfc-6160aa31ae05)


Подведем результаты:

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/671f87a0-de6e-40ce-a76f-5c67760ce204)

Значительное улучшение TPS относительно дефолтных настроек для EXT4. ZFS продолжает терять при увеличении объема данных.
XFS почти не почувствовала изменения.

## Эксперимент №3
>huge_pages = off
>
Со значением off большие страницы не будут запрашиваться.

EXT4

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/8cb91249-4af7-485f-8b01-57af9d9ae11b)

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/93b2f072-d493-4087-96a4-d623ca4e9332)

XFS

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/a95bd8b2-b5ce-4543-9f74-f8d1efe1e9d4)

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/0f371f01-376e-433d-92c8-213ad81f1826)

ZFS

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/25d45600-d404-469e-a316-3769717450aa)

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/0782c400-2051-446d-b7ff-68132765e026)


Подведем результаты:

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/2e6c144a-5f59-4d68-8f84-77df11e64179)

С данной конфигурацией XFS значительно улучшила показатели TPS и Latency для малого и большого объема данных

## Эксперимент №4
>random_page_cost = 1.0
>
Задаёт приблизительную стоимость чтения одной произвольной страницы с диска.
Значение PostgreSQL по умолчанию 4 для random_page_cost которое настроено для HDD. Мы используем SSD, потому поменяем его и проверим

EXT4

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/c88bf180-2708-4235-ba4b-665ab25d5096)

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/41288502-51ae-480c-8069-8b12a44919fb)

XFS

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/3d16d984-f5c3-4d67-9d11-dcc26cbfcc7a)

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/fd1942d8-1fd0-4247-bbf0-a471ce1b2395)

ZFS

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/d4dd95c1-3da5-4701-adad-55b6cf790f79)

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/4e9873f9-d934-4b25-a6ad-a612a9d9698e)

Подведем результаты:

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/f75a2b47-44e6-4663-b913-70d58490dbc7)

ZFS по-прежнему ухудшает показатели к увеличению объема данных.

## Эксперимент №5
Изменим кофигурацию:
![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/b5f8318b-2522-4693-b592-24129144ff58)

EXT4

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/f6e3bd08-1df9-4850-8ce4-02077037d418)
![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/ee99b30c-6593-4ef1-801c-b3918a10d68f)


XFS

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/725b02dc-86b4-4612-8c43-f8e928d41657)
![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/1ec7177c-5781-4410-944e-bb3b3fcac26a)

ZFS

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/defd1bf0-16a2-4c88-84c4-61fc1c5d6325)
![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/6daa67e3-fe3e-49e8-bae3-1fb44bbc06e5)


Подведем результаты:

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/57299e12-f7b4-47bf-98d0-a4cae896e026)

Заметим, что XFS к увеличению объема данных реагирует стабильнее и меньше теряет в производительности, TPS выше чем у EXT4. ZFS снова позади.

## Эксперимент №6
>synchronous_commit = off
>
Теперь отключим параметр synchronous_commit, чтобы сервер не сообщал об успешном выполнении операции

EXT4

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/5ee7fc8e-47e9-4857-a5df-d6c2beab9860)
![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/c5e5bd67-96ed-4d08-b3c1-16b9ffdde631)

XFS

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/c9a4c579-705a-400f-ad03-3b962e996d83)
![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/ed1a8749-41aa-4285-9f29-ad157c6b01d5)

ZFS

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/629158df-82fe-4b41-9580-ae8faacffda7)
![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/cb695129-def5-4b8a-8364-9f76ebfb73ae)


Подведем результаты:

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/b08041c6-09c3-4a28-9e7e-3974404699bf)

У EXT4 задержка получилась выше на большом объеме данных, нежели чем у XFS. ZFS снова проигрывает на обоих объемах данных.

## Эксперимент №7
Добавим настройки:
![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/5d94a139-5b52-4458-9564-5282e5d35fd4)

Первая настройка отключит прерывания на проверку записи данных на физический диск. Отключение второго параметра ускоряет обычные операции, 
но может привести к неисправимому повреждению или незаметной порче данных после сбоя системы. Так как при этом возникают практически те же риски, что и при отключении fsync, хотя и в меньшей степени, 
отключать его следует только при тех же обстоятельствах, которые перечислялись в рекомендациях для вышеописанного параметра.

EXT4

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/3c3722c2-6999-4b4a-b2cf-dc83af7ac310)

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/a202fc65-2c91-470c-aad6-bc2bb433a0e1)

XFS

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/b3ac9bbe-07ac-4eb0-8bb3-f3bffa7c112a)
![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/decbd7ca-c8fb-47d7-aa79-c0205ca9b635)

ZFS

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/49e2cbdf-4d4c-45d5-98df-1fdde4fcffaf)
![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/f2814dfe-1cca-41b5-88e4-28808bc6fd8d)



Подведем результаты:

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/eb59ae2d-b9f6-429f-a4f6-603e3b380db0)

Здесь XFS показала себя лучше, чем остальные на обоих объемах данных.

Итого общая таблица выглядит так:

![image](https://github.com/blaidex2/Postgres_Homework-Project/assets/130083589/24286b68-618b-4408-b706-e56e84f5f78b)




## Выводы
Рассмотрев поведение файловых систем при разных конфигурациях и объемах данных на основе нашего проекта сложно наверняка определить лучшую файловую систему для Postresql.
Можем сказать, что явный аутсайдер здесь файловая система ZFS  в плане производительности. При увеличении объема данных в 10 раз, она в 2-2.5 раза теряет свою производительность, что может критично сказаться
на онлайновых БД с большими объемами данных. Но зато более надежно распределяет данные и лучше всего подходит для восстановления после сбоя диска.
На основе данных, полученных на экспериментах, можно сказать, что система EXT4 больше подойдет для малых объемов данных, имея большую производительность среди конкурентов. 
Преимущество Ext4 перед другими системами заключается в его превосходной способности чтения и времени загрузки по сравнению с другими системами. Однако он не имеет расширенных функций, таких как прозрачное сжатие, и относительно медленнее записывает файлы.
Для достаточно грузных БД оптимальнее использовать XFS. Так как, когда дело доходит до файлов меньшего размера, XFS — не лучший вариант. Тем не менее, XFS компенсирует свои недостатки, обеспечивая лучшую поддержку для файлов большего размера по сравнению с конкурентами. XFS также поддерживает функции для твердотельных накопителей.
