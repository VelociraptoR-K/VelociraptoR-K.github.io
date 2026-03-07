# Лабораторная работа №9

## Цели работы:
1. Реализовать CRUD (Create, Read, Update, Delete) для сущностей бизнес-логики приложения.
2. Освоить работу с SQLite в памяти (:memory:) через модуль sqlite3.
3. Понять принципы первичных и внешних ключей и их роль в связях между таблицами.
4. Выделить контроллеры для работы с БД и для рендеринга страниц в отдельные модули.
5. Использовать архитектуру MVC и соблюдать разделение ответственности.
6. Отображать пользователям таблицу с валютами, на которые они подписаны.
7. Реализовать полноценный роутер, который обрабатывает GET-запросы и выполняет сохранение/обновление данных и рендеринг страниц.
8. Научиться тестировать функционал на примере сущностей currency и user с использованием unittest.mock.

## Задание
### Основные задачи:
CRUD для Currency
Create — добавление новых валют в базу данных.
Read — вывод валют из базы данных.
Update — обновление значения курса валюты.
Delete — удаление валюты по id.
Все действия должны использовать параметризованные запросы для защиты от SQL-инъекций.
Работа с SQLite
Использовать базу в памяти (sqlite3.connect(':memory:')).  
Объяснить, для чего нужны первичные ключи (PRIMARY KEY) и внешние ключи (FOREIGN KEY).
```sql
CREATE TABLE user (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL
);

CREATE TABLE currency (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    num_code TEXT NOT NULL,
    char_code TEXT NOT NULL,
    name TEXT NOT NULL,
    value FLOAT,
    nominal INTEGER
);

CREATE TABLE user_currency (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    currency_id INTEGER NOT NULL,
    FOREIGN KEY(user_id) REFERENCES user(id),
    FOREIGN KEY(currency_id) REFERENCES currency(id)
);
```

## Отрывок кода из лабораторной, демонстрирующий контроллер для CRUD операций с таблицей валют:

```python
import sqlite3
from typing import List, Dict, Any, Optional


class CurrencyRatesCRUD:
    """Контроллер для CRUD операций с таблицей валют."""

    def __init__(self, currency_rates_obj=None):
        """
        Инициализирует контроллер.

        Args:
            currency_rates_obj: объект с данными о валютах (для обратной совместимости)
        """
        self.__con = sqlite3.connect(':memory:')
        self.__cursor = self.__con.cursor()
        self.__currency_rates_obj = currency_rates_obj
        self.__createtable()

    def __createtable(self):
        """Создает таблицы в базе данных."""
        
        self.__con.execute("""
            CREATE TABLE currency (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                num_code TEXT NOT NULL,
                char_code TEXT NOT NULL,
                name TEXT NOT NULL,
                value FLOAT,
                nominal INTEGER
            )
        """)

        # Создаем таблицу пользователей для внешних ключей
        self.__con.execute("""
            CREATE TABLE user (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT NOT NULL
            )
        """)

        # Создаем таблицу связей пользователей с валютами
        self.__con.execute("""
            CREATE TABLE user_currency (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER NOT NULL,
                currency_id INTEGER NOT NULL,
                FOREIGN KEY(user_id) REFERENCES user(id),
                FOREIGN KEY(currency_id) REFERENCES currency(id)
            )
        """)

        self.__con.commit()

    def _create(self, currencies_data: List[Dict[str, Any]] = None):
        """
        Добавляет валюты в базу данных.

        Args:
            currencies_data: список словарей с данными о валютах.
                            Если None, использует данные из currency_rates_obj
        """
        if currencies_data is None and self.__currency_rates_obj:
            __params = self.__currency_rates_obj.values
            data = []
            for el in __params:
                if len(el) >= 5:
                    data.append({
                        "num_code": str(el[0]),  # Конвертируем в строку
                        "char_code": el[1],
                        "name": el[2],
                        "value": float(el[3]),
                        "nominal": int(el[4])
                    })
        else:
            data = currencies_data

        if data:
            __sqlquery = """
                INSERT INTO currency(num_code, char_code, name, value, nominal) 
                VALUES(:num_code, :char_code, :name, :value, :nominal)
            """
            self.__cursor.executemany(__sqlquery, data)
            self.__con.commit()

    def create_currency(self, num_code: str, char_code: str, name: str,
                        value: float, nominal: int) -> int:
        """
        Создает новую валюту в базе данных.

        Args:
            num_code: цифровой код валюты
            char_code: символьный код валюты
            name: название валюты
            value: курс валюты
            nominal: номинал

        Returns:
            int: ID созданной записи
        """
        query = """
            INSERT INTO currency(num_code, char_code, name, value, nominal) 
            VALUES(?, ?, ?, ?, ?)
        """
        self.__cursor.execute(query, (num_code, char_code, name, value, nominal))
        self.__con.commit()
        return self.__cursor.lastrowid

    def _read(self, currency_code: Optional[str] = None) -> List[Dict[str, Any]]:
        """
        Читает данные о валютах из базы данных.

        Args:
            currency_code: символьный код валюты для фильтрации (опционально)

        Returns:
            List[Dict[str, Any]]: список словарей с данными о валютах
        """
        if currency_code:
            if not isinstance(currency_code, str) or len(currency_code) != 3:
                raise ValueError("Код валюты должен быть строкой из 3 символов")
            cur = self.__con.execute(
                "SELECT * FROM currency WHERE char_code = ?",
                (currency_code,)
            )
        else:
            cur = self.__con.execute("SELECT * FROM currency ORDER BY char_code")

        result_data = []
        for _row in cur:
            _d = {
                'id': int(_row[0]),
                'num_code': _row[1],
                'char_code': _row[2],
                'name': _row[3],
                'value': float(_row[4]),
                'nominal': int(_row[5])
            }
            result_data.append(_d)
        return result_data

    def _delete(self, currency_id: int) -> bool:
        """
        Удаляет валюту по ID.

        Args:
            currency_id: ID валюты для удаления

        Returns:
            bool: True если удаление успешно
        """
        # Сначала удаляем связанные записи из user_currency
        self.__cursor.execute("DELETE FROM user_currency WHERE currency_id = ?", (currency_id,))
        # Затем удаляем валюту
        self.__cursor.execute("DELETE FROM currency WHERE id = ?", (currency_id,))
        self.__con.commit()
        return self.__cursor.rowcount > 0

    def _update(self, currency: Dict[str, float]) -> bool:
        """
        Обновляет курс валюты.

        Args:
            currency: словарь {код_валюты: новое_значение}

        Returns:
            bool: True если обновление успешно
        """
        currency_code = list(currency.keys())[0]
        currency_value = list(currency.values())[0]

        self.__cursor.execute(
            "UPDATE currency SET value = ? WHERE char_code = ?",
            (currency_value, currency_code)
        )
        self.__con.commit()
        return self.__cursor.rowcount > 0

    def update_currency_value(self, currency_id: int, new_value: float) -> bool:
        """
        Обновляет курс валюты по ID.

        Args:
            currency_id: ID валюты
            new_value: новое значение курса

        Returns:
            bool: True если обновление успешно
        """
        self.__cursor.execute(
            "UPDATE currency SET value = ? WHERE id = ?",
            (new_value, currency_id)
        )
        self.__con.commit()
        return self.__cursor.rowcount > 0

    def __del__(self):
        self.__cursor = None
        self.__con.close()
```
## Выводы:
1. Применение архитектуры MVC:

- Model: Классы в папке models представляют бизнес-сущности и их логику

- View: Шаблоны в папке templates отвечают за представление данных

- Controller: Классы в папке controllers управляют бизнес-логикой и взаимодействием с БД

Преимущества:

- Четкое разделение ответственности

- Легкость поддержки и модификации

2. Работа с SQLite:
- Использована база данных в памяти (:memory:)

- Реализованы параметризованные запросы для защиты от SQL-инъекций

- Созданы таблицы с первичными и внешними ключами

- Обеспечена целостность данных через связи между таблицами

Первичные ключи (PRIMARY KEY):

- Уникально идентифицируют каждую запись

- Автоматически инкрементируются (AUTOINCREMENT)

- Используются для связей с другими таблицами

Внешние ключи (FOREIGN KEY):

- Обеспечивают ссылочную целостность данных

- Запрещают удаление записей, на которые есть ссылки

- Реализуют связи между таблицами

3. Обработка маршрутов
- Использован BaseHTTPRequestHandler для обработки HTTP-запросов

- Реализована маршрутизация на основе URL-путей

- Поддержка GET-параметров через parse_qs

- Обработка ошибок (404 страница)

Примеры маршрутов:

- / - главная страница

- /currencies - список валют

- /currency/update?USD=9999.67 - обновление курса

- /currency/delete?id=3 - удаление валюты

4. Рендеринг шаблонов
- Использован шаблонизатор Jinja2

- Передача данных из контроллера в представление

- Наследование шаблонов и повторное использование компонентов

- Проект демонстрирует полный цикл разработки веб-приложения с использованием современных подходов к безопасности, тестированию и архитектуре разделения файлов.

### С полным содержанием Лабораторной работы 9 можно ознакомиться по [ссылке:](https://github.com/VelociraptoR-K/Repo_Py/tree/main/Lab_9)