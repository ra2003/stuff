# tabun_stat

Считалка статистики для [Табуна](https://tabun.everypony.ru/). На вход
(который нужно написать самостоятельно) принимает пользователей, блоги, посты
и комменты, а на выходе даёт пачку файлов (в основном csv) с разными
интересными числами и иногда словами.

Статистику 2011-2018 с красивыми графиками можно посмотреть здесь:
https://tabun.everypony.ru/blog/uniblog/182474.html


## Установка и использование

Стандартно, скачав и запустив в каталоге tabun_stat:

    pip install .

После этого появится команда `tabun_stat`

Для работы требуется конфиг; вы можете найти пример в `config.example.toml`.
Сохраните его как, например, `config.toml`, отредактируйте его для своих нужд
и запустите следующей командой:

    tabun_stat -vv -c config.toml

Опция `-vv` включает подробный вывод с прогресс-барами.

Для работы tabun_stat требуется какой-то источник данных. Подразумевается,
что он у вас есть и вы его можете подключить самостоятельно. В репозитории
лежит демонстрационный пример данных для sqlite3 базы данных; чтобы
воспользоваться им, создайте базу из прилагаемого sql-дампа:

    $ sqlite3 demo.sqlite3 < demo.sql

и включите sqlite3 источник данных в конфиг-файле:

    [datasource]
    name = ":sqlite3.Sqlite3DataSource"
    path = "demo.sqlite3"

(двоеточие в начале name — сокращение для поиска модуля внутри пакета
`tabun_stat.datasource`.)


## Как создать свой источник данных

Источник данных — это класс-потомок `tabun_stat.datasource.BaseDataSource.`
Чтобы создать свой источник, унаследуйтесь от него, реализуйте все абстрактные
методы:

    from tabun_stat.datasource import BaseDataSource

    class CustomDataSource(BaseDataSource):
        def __init__(self, option1: str, option2: int) -> None:
            self.option1 = option1
            self.option2 = option2
            # и так далее по вкусу

        def destroy(self) -> None:
            pass  # по вкусу

        # и так далее

и включите его использование в конфиге:

    [datasource]
    name = "ваш_модуль.CustomDataSource"
    option1 = "значение опции 1"
    option2 = 777

Все ключи из конфига, кроме `name`, передаются в `__init__` как аргументы.

Кратко о том, какие абстрактные методы нужно переопределить:

* `get_user_by_id` и `get_user_by_name` — получение одного пользователя;

* `get_users_limits` — получение статистики по пользователям;

* `get_blog_by_id` и `get_blog_by_slug` — получение одного блога;

* `get_blogs_limits` — получение статистики по блогам;

* `get_post_by_id` — получение одного поста;

* `get_posts_limits` — получение статистики по постам;

* `get_post_comments` — получение всех комментов для указанного поста;

* `get_comment_by_id` — получение одного коммента;

* `get_comments_limits` — получение статистики по комментам.

Для источников на базе популярных СУБД почти всё это сводится к несложным
SQL-запросам.

Также есть ещё методы, имеющие реализацию по умолчанию, но крайне
неэффективную (в сотни раз медленнее эффективных реализаций на SQL),
поэтому их тоже крайне желательно переопределить:

* `get_username_by_user_id`, `get_blog_status_by_id`,
  `get_blog_status_by_slug`, `get_blog_id_by_slug` и `get_blog_id_of_post` —
  для этих методов можно применить какое-нибудь кэширование;

* `iter_users` — получение всех пользователей по очереди;

* `iter_blogs` — получение всех блогов;

* `iter_posts` — получение всех постов;

* `iter_comments` — получение всех комментов.

iter-методы — это генераторы, которые выдают запрашиваемые объекты через yield.
Если указаны фильтры, то должны применяться прописанные в них ограничения.
Сортировка возвращаемых значений не определена. В целях оптимизации yield'ятся
не отдельно объекты по одному, а списки объектов. (В стандартной неэффективной
реализации этот список состоит из одного элемента, а лучше бы сотни.)

Есть ещё несколько методов, стандартная реализация которых достаточно
эффективна, но иногда может быть полезно переопределить и их тоже (например,
чтобы прикрутить кэширование):

* `get_username_by_user_id` — получить имя пользователя по его id;

* `get_blog_status_by_id` и `get_blog_status_by_slug` — статус блога по его id
  или slug;

* `get_blog_id_by_slug` — перевод slug блога в id блога;

* `get_blog_id_of_post` — получение блога поста.

Подробнее о том, как это всё реализовывать, читайте в docstring'ах в файле
`tabun_stat/processors/base.py`. И вообще, код — лучшая документация,
читайте пример для sqlite3 ;)


## Как создать свой обработчик

Обработчик — это класс-потомок `tabun_stat.processors.BaseProcessor`.
Чтобы создать свой источник, унаследуйтесь от него и реализуйте нужные вам
методы на свой вкус.

Каждый обработчик считает полную статистику ровно один раз, после чего
уничтожается. Использование одного и того же обработчика для подсчёта
полной статистики несколько раз не предусмотрено.

Жизненный цикл обработчика (какие методы вызываются):

* сперва он просто создаётся без привязки к настройкам или источнику;

* `start` — принимает объект со статистикой и охватываемый период. Между
  `start` и `stop` доступен объект `self.stat`, через который можно получить
  доступ к источнику данных `self.stat.source` (публичные методы источника
  описаны выше);

* `begin_users`;

* `process_user` вызывается столько раз, сколько есть пользователей
  (сортировка пользователей не определена);

* `end_users`;

* `begin_blogs`;

* `process_blog` вызывается столько раз, сколько есть блогов (сортировка
  блогов не определена);

* `end_blogs`;

* `begin_messages`;

* `process_post` и `process_comment` — вызываются гарантированно с сортировкой
  постов и комментов по времени, что позволяет сделать некоторые оптимизации;

* `end_messages`;

* `stop` — здесь `self.stat` отцепляется;

* после всего этого обработчики уничтожаются.

Не забывайте про `super`:
    
    import typing
    from typing import Optional, Dict, Any
    from datetime import datetime

    from tabun_stat.stat import TabunStat
    from tabun_stat.processors import BaseProcessor

    class CustomProcessor(BaseProcessor):
        def __init__(self, option1: str, option2: int) -> None:
            self.option1 = option1
            self.option2 = option2
            # и так далее по вкусу

        def start(
            self, stat: TabunStat,
            min_date: Optional[datetime] = None, max_date: Optional[datetime] = None
        ) -> None:
            super().start(stat, min_date, max_date)
            # что-то там

        def process_user(self, user: Dict[str, Any]) -> None:
            ... # что-то там

        def stop(self) -> None:
            # что-то там
            super().stop()

        # и так далее

Включается в конфиге аналогично источнику. Обратите внимание, что `processors`
это массив, и поэтому в синтаксисе TOML используются две пары квадратных
скобок:

    [[processors]]
    name = "custom_module.CustomProcessor"
    option1 = "hello"
    option2 = 4

    [[processors]]
    name = ":registrations.RegistrationsProcessor"
    # ...и другие стандартные обработчики по желанию

Все ключи из конфига, кроме `name`, передаются в `__init__` как аргументы.
