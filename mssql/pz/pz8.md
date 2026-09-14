# Практическое задание

# Проектирование и реализация представлений в MS SQL Server

## Цель

Спроектировать и реализовать набор представлений в MS SQL Server для упрощения доступа к данным, сокрытия избыточной структуры таблиц, ограничения набора доступных столбцов и подготовки данных для типовых пользовательских и аналитических запросов.

В рамках практической работы необходимо создать несколько связанных таблиц, разработать простые и сложные представления, выполнить изменение и удаление представлений, проверить возможность выполнения операций через представление и исследовать влияние представлений на производительность запросов.

Отдельное внимание необходимо уделить тому, что обычное представление само по себе не хранит результат запроса как отдельный набор данных. При обращении к нему SQL Server формирует план выполнения на основе запроса представления и запроса, обращающегося к этому представлению.

Основной результат работы — получить структуру представлений, которая делает работу с базой данных удобнее, но при этом не скрывает проблемы неэффективных запросов и неправильного проектирования.

# Задание

Необходимо создать базу данных для системы обработки заказов.

База должна содержать информацию о клиентах, товарах, категориях, заказах и позициях заказов.

На основе этих данных необходимо разработать несколько представлений разного назначения.

Минимально необходимо создать:

- простое представление;
- представление с соединением нескольких таблиц;
- представление с вычисляемыми столбцами;
- представление с агрегированием;
- представление для ограничения доступа к данным;
- представление с фильтрацией;
- представление, через которое будет проверена возможность изменения данных.

---

# 1. Создание базы данных

Создать новую базу данных:

```sql
CREATE DATABASE ViewsPracticeDB;
GO

USE ViewsPracticeDB;
GO
```

Создать отдельную схему:

```sql
CREATE SCHEMA Sales;
GO
```

Все основные таблицы разместить в этой схеме.

---

# 2. Создание таблицы клиентов

Создать таблицу:

```sql
CREATE TABLE Sales.Customers
(
    CustomerId INT IDENTITY(1,1)
        CONSTRAINT PK_Customers PRIMARY KEY,

    FullName NVARCHAR(200) NOT NULL,

    Email NVARCHAR(200) NOT NULL,

    Phone NVARCHAR(30) NULL,

    City NVARCHAR(100) NOT NULL,

    RegistrationDate DATE NOT NULL,

    IsActive BIT NOT NULL
        CONSTRAINT DF_Customers_IsActive DEFAULT 1
);
```

Добавить ограничение уникальности:

```sql
ALTER TABLE Sales.Customers
ADD Constraint UQ_CUSTOMERS_EMAIL
UNIQUE (Email);
```

---

# 3. Создание таблицы категорий

Создать таблицу:

```sql
CREATE TABLE Sales.Categories
(
    CategoryId INT IDENTITY(1,1)
        CONSTRAINT PK_Categories PRIMARY KEY,

    CategoryName NVARCHAR(100) NOT NULL,

    IsActive BIT NOT NULL
        CONSTRAINT DF_Categories_IsActive DEFAULT 1
);
```

---

# 4. Создание таблицы товаров

Создать таблицу:

```sql
CREATE TABLE Sales.Products
(
    ProductId INT IDENTITY(1,1)
        CONSTRAINT PK_Products PRIMARY KEY,

    CategoryId INT NOT NULL,

    ProductName NVARCHAR(200) NOT NULL,

    Price DECIMAL(12,2) NOT NULL,

    StockQuantity INT NOT NULL,

    IsActive BIT NOT NULL
        CONSTRAINT DF_Products_IsActive DEFAULT 1,

    CONSTRAINT FK_Products_Categories
        FOREIGN KEY (CategoryId)
        REFERENCES Sales.Categories(CategoryId),

    CONSTRAINT CK_Products_Price
        CHECK (Price >= 0),

    CONSTRAINT CK_Products_StockQuantity
        CHECK (StockQuantity >= 0)
);
```

---

# 5. Создание таблицы заказов

Создать таблицу:

```sql
CREATE TABLE Sales.Orders
(
    OrderId INT IDENTITY(1,1)
        CONSTRAINT PK_Orders PRIMARY KEY,

    CustomerId INT NOT NULL,

    OrderDate DATETIME2 NOT NULL
        CONSTRAINT DF_Orders_OrderDate DEFAULT SYSDATETIME(),

    Status NVARCHAR(30) NOT NULL,

    ManagerId INT NULL,

    CONSTRAINT FK_Orders_Customers
        FOREIGN KEY (CustomerId)
        REFERENCES Sales.Customers(CustomerId),

    CONSTRAINT CK_Orders_Status
        CHECK
        (
            Status IN
            (
                N'Новый',
                N'В обработке',
                N'Завершён',
                N'Отменён'
            )
        )
);
```

---

# 6. Создание таблицы позиций заказа

Создать таблицу:

```sql
CREATE TABLE Sales.OrderItems
(
    OrderItemId INT IDENTITY(1,1)
        CONSTRAINT PK_OrderItems PRIMARY KEY,

    OrderId INT NOT NULL,

    ProductId INT NOT NULL,

    Quantity INT NOT NULL,

    UnitPrice DECIMAL(12,2) NOT NULL,

    CONSTRAINT FK_OrderItems_Orders
        FOREIGN KEY (OrderId)
        REFERENCES Sales.Orders(OrderId),

    CONSTRAINT FK_OrderItems_Products
        FOREIGN KEY (ProductId)
        REFERENCES Sales.Products(ProductId),

    CONSTRAINT CK_OrderItems_Quantity
        CHECK (Quantity > 0),

    CONSTRAINT CK_OrderItems_UnitPrice
        CHECK (UnitPrice >= 0)
);
```

---

# 7. Заполнение таблиц данными

Добавить тестовые данные.

Минимальный объем:

```text
Customers — 20 записей
Categories — 5 записей
Products — 30 записей
Orders — 50 записей
OrderItems — 100 записей
```

Данные должны включать:

```text
активных и неактивных клиентов;
несколько городов;
разные категории товаров;
активные и неактивные товары;
заказы разных статусов;
заказы за разные даты;
несколько товаров внутри одного заказа.
```

При заполнении необходимо использовать реалистичные данные.

---

# 8. Создание первого простого представления

Создать представление, которое показывает активных клиентов.

```sql
CREATE VIEW Sales.vw_ActiveCustomers
AS
SELECT
    CustomerId,
    FullName,
    Email,
    Phone,
    City,
    RegistrationDate
FROM Sales.Customers
WHERE IsActive = 1;
GO
```

Проверить результат:

```sql
SELECT *
FROM Sales.Vw_activecustomers;
```

Определить, чем результат запроса через представление отличается от прямого запроса к таблице.

---

# 9. Создание представления для сокрытия данных

Создать представление, которое предоставляет ограниченную информацию о клиентах.

Например:

```sql
CREATE VIEW Sales.vw_CustomersPublic
AS
SELECT
    CustomerId,
    FullName,
    City,
    RegistrationDate
FROM Sales.Customers;
GO
```

В представление намеренно не включать:

```text
Email
Phone
IsActive
```

Проверить:

```sql
SELECT *
FROM Sales.Vw_customerspublic;
```

Определить, какие данные теперь недоступны через представление.

---

# 10. Создание представления с JOIN

Создать представление каталога товаров с категориями.

```sql
CREATE VIEW Sales.vw_ProductCatalog
AS
SELECT
    p.ProductId,
    p.ProductName,
    c.CategoryName,
    p.Price,
    p.StockQuantity,
    p.IsActive
FROM Sales.Products AS p
INNER JOIN Sales.Categories AS c
    ON c.CategoryId = p.CategoryId;
GO
```

Проверить:

```sql
SELECT *
FROM Sales.Vw_productcatalog;
```

Затем выполнить фильтрацию уже через представление:

```sql
SELECT
    PRODUCTID,
    PRODUCTNAME,
    CATEGORYNAME,
    PRICE
FROM SALES.VW_PRODUCTCATALOG
WHERE PRICE > 5000;
```

Обратить внимание, что дополнительное условие задается уже при обращении к представлению.

---

# 11. Создание представления с несколькими JOIN

Создать представление с информацией о заказах.

Оно должно объединять:

```text
Orders
Customers
OrderItems
Products
```

Пример:

```sql
CREATE VIEW Sales.vw_OrderDetails
AS
SELECT
    o.OrderId,
    o.OrderDate,
    o.Status,
    c.CustomerId,
    c.FullName AS CustomerName,
    p.ProductId,
    p.ProductName,
    oi.Quantity,
    oi.UnitPrice,
    oi.Quantity * oi.UnitPrice AS LineTotal
FROM Sales.Orders AS o
INNER JOIN Sales.Customers AS c
    ON c.CustomerId = o.CustomerId
INNER JOIN Sales.OrderItems AS oi
    ON oi.OrderId = o.OrderId
INNER JOIN Sales.Products AS p
    ON p.ProductId = oi.ProductId;
GO
```

Проверить результат:

```sql
SELECT *
FROM Sales.Vw_orderdetails;
```

Затем получить только завершенные заказы:

```sql
SELECT *
FROM Sales.Vw_orderdetails
WHERE Status = N 'Завершён';
```

---

# 12. Создание представления с вычисляемым столбцом

Создать представление, которое рассчитывает сумму каждой позиции заказа.

Если такой расчет уже присутствует в предыдущем представлении, создать отдельное представление специально для этого эксперимента.

Например:

```sql
CREATE VIEW Sales.vw_OrderItemAmounts
AS
SELECT
    OrderItemId,
    OrderId,
    ProductId,
    Quantity,
    UnitPrice,
    Quantity * UnitPrice AS Amount
FROM Sales.OrderItems;
GO
```

Проверить:

```sql
SELECT *
FROM Sales.Vw_orderitemamounts;
```

Выполнить запрос:

```sql
SELECT
    ORDERID,
    SUM(AMOUNT) AS ORDERAMOUNT
FROM SALES.VW_ORDERITEMAMOUNTS
GROUP BY ORDERID;
```

---

# 13. Создание агрегированного представления

Создать представление с общей суммой каждого заказа.

```sql
CREATE VIEW Sales.vw_OrderTotals
AS
SELECT
    o.OrderId,
    o.CustomerId,
    o.OrderDate,
    o.Status,
    SUM(oi.Quantity * oi.UnitPrice) AS TotalAmount
FROM Sales.Orders AS o
INNER JOIN Sales.OrderItems AS oi
    ON oi.OrderId = o.OrderId
GROUP BY
    o.OrderId,
    o.CustomerId,
    o.OrderDate,
    o.Status;
GO
```

Проверить:

```sql
SELECT *
FROM Sales.Vw_ordertotals;
```

Получить заказы дороже заданного значения:

```sql
SELECT *
FROM Sales.Vw_ordertotals
WHERE Totalamount > 50000;
```

---

# 14. Создание представления для аналитической отчетности

Создать представление, которое показывает продажи по клиентам.

Оно должно содержать:

```text
CustomerId
FullName
OrdersCount
TotalPurchased
AverageOrderAmount
```

Пример основы:

```sql
CREATE VIEW Sales.vw_CustomerSalesSummary
AS
SELECT
    c.CustomerId,
    c.FullName,
    COUNT(DISTINCT o.OrderId) AS OrdersCount,
    SUM(oi.Quantity * oi.UnitPrice) AS TotalPurchased
FROM Sales.Customers AS c
INNER JOIN Sales.Orders AS o
    ON o.CustomerId = c.CustomerId
INNER JOIN Sales.OrderItems AS oi
    ON oi.OrderId = o.OrderId
GROUP BY
    c.CustomerId,
    c.FullName;
GO
```

При необходимости расчет средней суммы заказа можно выполнить при обращении к представлению либо изменить структуру представления.

Проверить:

```sql
SELECT *
FROM Sales.Vw_customersalessummary
ORDER BY Totalpurchased DESC;
```

---

# 15. Изменение существующего представления

Изменить:

```text
Sales.vw_ActiveCustomers
```

Добавить дополнительное поле:

```text
IsActive
```

Использовать:

```sql
ALTER VIEW Sales.vw_ActiveCustomers
AS
SELECT
    CustomerId,
    FullName,
    Email,
    Phone,
    City,
    RegistrationDate,
    IsActive
FROM Sales.Customers
WHERE IsActive = 1;
GO
```

Проверить результат.

---

# 16. Использование CREATE OR ALTER VIEW

Создать или изменить представление через:

```sql
CREATE OR ALTER VIEW
```

Например:

```sql
CREATE OR ALTER VIEW Sales.vw_ProductCatalog
AS
SELECT
    p.ProductId,
    p.ProductName,
    c.CategoryName,
    p.Price,
    p.StockQuantity
FROM Sales.Products AS p
INNER JOIN Sales.Categories AS c
    ON c.CategoryId = p.CategoryId
WHERE p.IsActive = 1;
GO
```

Проверить, что в результате отображаются только активные товары.

---

# 17. Проверка возможности INSERT через представление

Создать простое представление:

```sql
CREATE VIEW Sales.vw_CustomerInput
AS
SELECT
    FullName,
    Email,
    Phone,
    City,
    RegistrationDate,
    IsActive
FROM Sales.Customers;
GO
```

Попытаться добавить данные через представление:

```sql
INSERT INTO Sales.Vw_customerinput
(
    Fullname,
    Email,
    Phone,
    City,
    Registrationdate,
    Isactive
)
VALUES
(
    N 'Алексей Соколов',
    N 'alexey.sokolov@example.com',
    N '+7 900 000-00-00',
    N 'Москва',
    '2026-09-01',
    1
);
```

После этого проверить таблицу:

```sql
SELECT *
FROM Sales.Customers
WHERE Email = N 'alexey.sokolov@example.com';
```

Зафиксировать результат.

---

# 18. Проверка UPDATE через представление

Выполнить изменение:

```sql
UPDATE Sales.Vw_customerinput
SET City = N 'Казань'
WHERE Email = N 'alexey.sokolov@example.com';
```

Проверить изменение в основной таблице:

```sql
SELECT
    FULLNAME,
    EMAIL,
    CITY
FROM SALES.CUSTOMERS
WHERE EMAIL = N 'alexey.sokolov@example.com';
```

Сделать вывод, что изменение данных через представление в некоторых случаях возможно.

---

# 19. Проверка изменения данных через сложное представление

Попытаться выполнить изменение через представление:

```text
Sales.vw_OrderDetails
```

Например, изменить одновременно данные, относящиеся к разным исходным таблицам.

Проверить реакцию SQL Server.

Зафиксировать ошибку или результат выполнения.

Определить, почему операции изменения через сложные представления имеют ограничения.

---

# 20. Создание представления с CHECK OPTION

Создать представление:

```sql
CREATE VIEW Sales.vw_MoscowCustomers
AS
SELECT
    CustomerId,
    FullName,
    Email,
    City,
    IsActive
FROM Sales.Customers
WHERE City = N'Москва'
WITH CHECK OPTION;
GO
```

Проверить:

```sql
SELECT *
FROM Sales.Vw_moscowcustomers;
```

Попытаться изменить город через представление:

```sql
UPDATE Sales.Vw_moscowcustomers
SET City = N 'Казань'
WHERE Customerid = 1;
```

Зафиксировать результат.

После этого объяснить назначение:

```sql
WITH CHECK OPTION
```

---

# 21. Создание представления WITH SCHEMABINDING

Создать простое представление с привязкой к схеме.

Например:

```sql
CREATE VIEW Sales.vw_ProductsBound
WITH SCHEMABINDING
AS
SELECT
    ProductId,
    ProductName,
    Price,
    StockQuantity
FROM Sales.Products;
GO
```

После этого попытаться изменить структуру исходной таблицы таким образом, чтобы изменение затрагивало используемый представлением столбец.

Например, попытаться удалить:

```text
ProductName
```

Зафиксировать результат.

Сделать вывод о назначении `SCHEMABINDING`.

---

# 22. Просмотр информации о представлениях

Получить список представлений:

```sql
SELECT
    name,
    object_id,
    create_date,
    modify_date
FROM sys.views;
```

Дополнительно получить информацию только по схеме:

```sql
SELECT
    SCHEMA_NAME(schema_id) AS schemaname,
    name AS viewname,
    create_date,
    modify_date
FROM sys.views
WHERE schema_id = SCHEMA_ID('Sales');
```

---

# 23. Просмотр определения представления

Получить определение представления:

```sql
EXEC sp_helptext 'Sales.vw_OrderDetails';
```

Также изучить:

```sql
SELECT OBJECT_DEFINITION(
    OBJECT_ID('Sales.vw_OrderDetails')
);
```

Сравнить способы получения текста определения.

---

# 24. Анализ зависимостей представления

Исследовать зависимости представления.

Например:

```sql
SELECT
    referenced_schema_name,
    referenced_entity_name
FROM sys.dm_sql_referenced_entities(
    'Sales.vw_OrderDetails',
    'OBJECT'
);
```

Определить, какие таблицы используются данным представлением.

---

# 25. Сравнение прямого запроса и запроса через представление

Сначала выполнить прямой запрос:

```sql
SELECT
    o.orderid,
    o.orderdate,
    c.fullname,
    p.productname,
    oi.quantity,
    oi.unitprice,
    oi.quantity * oi.unitprice AS linetotal
FROM sales.orders AS o
INNER JOIN sales.customers AS c
    ON c.customerid = o.customerid
INNER JOIN sales.orderitems AS oi
    ON oi.orderid = o.orderid
INNER JOIN sales.products AS p
    ON p.productid = oi.productid
WHERE o.status = N 'Завершён';
```

После этого выполнить эквивалентный запрос через представление:

```sql
SELECT
    ORDERID,
    ORDERDATE,
    CUSTOMERNAME,
    PRODUCTNAME,
    QUANTITY,
    UNITPRICE,
    LINETOTAL
FROM SALES.VW_ORDERDETAILS
WHERE STATUS = N 'Завершён';
```

Включить:

```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
```

Получить фактические планы выполнения обоих запросов.

Сравнить:

```text
Logical Reads
CPU Time
Elapsed Time
Операторы плана
Estimated Rows
Actual Rows
```

Определить, создается ли принципиально другой способ получения данных только из-за использования обычного представления.

---

# 26. Проверка фильтрации через представление

Выполнить:

```sql
SELECT *
FROM Sales.Vw_orderdetails
WHERE Customerid = 10;
```

Получить фактический план.

После этого выполнить аналогичный запрос напрямую через таблицы.

Сравнить планы выполнения.

Задача — проверить, как оптимизатор обрабатывает дополнительное условие, указанное при обращении к представлению.

---

# 27. Проверка представления с SELECT \*

Создать временное экспериментальное представление:

```sql
CREATE VIEW Sales.vw_ProductsBadExample
AS
SELECT *
FROM Sales.Products;
GO
```

После создания проанализировать этот подход.

Определить возможные недостатки использования:

```sql
SELECT *
```

в определениях представлений.

После эксперимента удалить представление:

```sql
DROP VIEW Sales.vw_ProductsBadExample;
GO
```

---

# 28. Проверка вложенных представлений

Создать представление на основе другого представления.

Например:

```sql
CREATE VIEW Sales.vw_ExpensiveProducts
AS
SELECT
    ProductId,
    ProductName,
    CategoryName,
    Price
FROM Sales.vw_ProductCatalog
WHERE Price >= 10000;
GO
```

Проверить:

```sql
SELECT *
FROM Sales.Vw_expensiveproducts;
```

Получить фактический план выполнения.

Определить, какие базовые таблицы участвуют в запросе.

Сделать вывод о том, стоит ли создавать длинные цепочки вложенных представлений без необходимости.

---

# 29. Создание представления с фильтрацией по статусу

Создать представление:

```sql
CREATE VIEW Sales.vw_CompletedOrders
AS
SELECT
    OrderId,
    CustomerId,
    OrderDate,
    Status
FROM Sales.Orders
WHERE Status = N'Завершён';
GO
```

Проверить количество строк:

```sql
SELECT COUNT(*)
FROM Sales.Vw_completedorders;
```

Сравнить с прямым запросом:

```sql
SELECT COUNT(*)
FROM Sales.Orders
WHERE Status = N 'Завершён';
```

---

# 30. Проверка производительности без подходящих индексов

Выбрать представление, которое активно использует:

```text
CustomerId
OrderDate
Status
```

Получить план выполнения до создания дополнительных индексов.

Например:

```sql
SELECT *
FROM Sales.Vw_ordertotals
WHERE Customerid = 10;
```

Зафиксировать:

```text
Logical Reads
операторы Scan/Seek
CPU Time
```

---

# 31. Создание индекса на базовой таблице

Создать подходящий индекс на исходной таблице.

Например:

```sql
CREATE NONCLUSTERED INDEX IX_Orders_CustomerId
ON Sales.Orders(CustomerId)
INCLUDE
(
    OrderDate,
    Status
);
```

Повторить запрос через представление:

```sql
SELECT *
FROM Sales.Vw_ordertotals
WHERE Customerid = 10;
```

Сравнить план выполнения до и после создания индекса.

Сделать вывод о том, что производительность обычного представления во многом определяется структурой и индексами исходных таблиц.

---

# 32. Проверка производительности агрегированного представления

Выполнить:

```sql
SELECT *
FROM Sales.Vw_customersalessummary;
```

Зафиксировать план выполнения.

Обратить внимание на:

```text
Hash Match
Stream Aggregate
Sort
Join
Scan
Seek
```

Определить наиболее дорогие части плана.

Проверить, можно ли улучшить выполнение за счет индексов на исходных таблицах.

---

# 33. Эксперимент с избыточно сложным представлением

Создать временное представление, которое объединяет несколько таблиц и возвращает значительно больше столбцов, чем требуется конкретному запросу.

После этого выполнить запрос, выбирающий только несколько столбцов.

Получить фактический план.

Сравнить его с более простым представлением.

Задача — определить, когда универсальные представления становятся сложнее для сопровождения и анализа.

После эксперимента временное представление удалить.

---

# 34. Управление представлениями

В ходе работы необходимо использовать все основные операции управления.

Создание:

```sql
CREATE VIEW
```

Изменение:

```sql
ALTER VIEW
```

Создание или изменение:

```sql
CREATE OR ALTER VIEW
```

Удаление:

```sql
DROP VIEW
```

Получение определения:

```sql
OBJECT_DEFINITION
```

Просмотр списка:

```sql
sys.views
```

---

# 35. Разработка собственного набора представлений

После выполнения учебных примеров самостоятельно разработать минимум три дополнительных представления.

Первое должно быть предназначено для операционной работы.

Например:

```text
текущие активные заказы
```

Второе — для отчетности.

Например:

```text
продажи по категориям
```

Третье — для ограничения предоставляемых данных.

Например:

```text
информация о клиентах без контактных данных
```

Для каждого представления необходимо указать его назначение.

---

# 36. Подготовка итоговой таблицы представлений

Сформировать результат:

| Представление           | Назначение         | Исходные таблицы     | JOIN | Агрегация | Возможность изменения |
| ----------------------- | ------------------ | -------------------- | ---- | --------- | --------------------- |
| vw_ActiveCustomers      | Активные клиенты   | Customers            | Нет  | Нет       | Проверить             |
| vw_ProductCatalog       | Каталог            | Products, Categories | Да   | Нет       | Проверить             |
| vw_OrderDetails         | Детали заказов     | 4 таблицы            | Да   | Нет       | Ограничена            |
| vw_OrderTotals          | Суммы заказов      | Orders, OrderItems   | Да   | Да        | Ограничена            |
| vw_CustomerSalesSummary | Аналитика клиентов | Несколько            | Да   | Да        | Ограничена            |

Таблицу дополнить собственными представлениями.

---

# Подсказки по ключевым частям

Обычное представление не следует воспринимать как отдельную копию данных. Оно хранит определение запроса. Когда выполняется:

```sql
SELECT *
FROM Sales.Vw_productcatalog;
```

SQL Server анализирует определение представления и строит план обращения к базовым объектам.

Поэтому создание обычного представления не означает автоматического ускорения запроса.

Если исходные таблицы имеют неудачную структуру или отсутствуют необходимые индексы, представление само по себе эту проблему не исправит.

Представление удобно использовать как слой абстракции.

Например, вместо постоянного повторения:

```sql
FROM Sales.Products AS p
INNER JOIN Sales.Categories AS c
    ON c.CategoryId = p.CategoryId
```

можно предоставить пользователю готовое:

```sql
Sales.vw_ProductCatalog
```

Это уменьшает повторение SQL-кода и скрывает часть внутренней структуры базы данных.

Представления также можно применять для ограничения набора предоставляемых столбцов. Например, приложение может получать имя клиента и город, но не иметь необходимости работать с электронной почтой или телефоном.

При этом само наличие представления не заменяет полноценную систему разрешений SQL Server. Если пользователь имеет прямой доступ к исходной таблице, он все равно сможет получить данные из нее. В реальной системе представления необходимо рассматривать вместе с правами доступа.

При использовании `WITH CHECK OPTION` необходимо учитывать фильтр представления.

Если представление показывает только:

```sql
WHERE City = N'Москва'
```

то изменение строки через такое представление не должно приводить к ситуации, когда измененная строка перестает удовлетворять условию представления.

`SCHEMABINDING` связывает определение представления со структурой используемых объектов. Это полезно там, где изменение исходной таблицы не должно незаметно нарушить зависимый объект.

Необходимо избегать чрезмерной вложенности представлений.

Структура вида:

```text
представление использует другое представление;
оно использует еще одно;
третье использует четвертое;
```

может значительно усложнить анализ реального SQL-запроса и плана выполнения.

Также желательно избегать:

```sql
SELECT *
```

в определении долгоживущих представлений.

Лучше явно указывать нужные столбцы:

```sql
SELECT
    PRODUCTID,
    PRODUCTNAME,
    PRICE
FROM SALES.PRODUCTS;
```

Так структура интерфейса представления становится контролируемой и понятной.

При анализе производительности необходимо сравнивать эквивалентные запросы. Если прямой запрос возвращает пять столбцов, а представление двадцать, сравнение будет некорректным.

Используйте:

```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
```

и фактический план выполнения.

Не делайте вывод о производительности только по одному значению времени. На локальной тестовой базе время может сильно зависеть от кеширования.

В обычных представлениях основная оптимизация чаще всего выполняется через:

```text
структуру базовых таблиц;
правильные типы данных;
условия JOIN;
фильтрацию;
индексы;
статистику.
```

---

# Что проверить перед отправкой — чек-лист

## База данных

☐ Создана отдельная база данных.

☐ Создана рабочая схема.

☐ Созданы все необходимые таблицы.

☐ Таблицы связаны внешними ключами.

☐ Добавлены тестовые данные.

---

## Простые представления

☐ Создано представление активных клиентов.

☐ Создано представление с ограниченным набором столбцов.

☐ Проверено получение данных через представления.

---

## Сложные представления

☐ Создано представление с `JOIN`.

☐ Создано представление с несколькими `JOIN`.

☐ Создано представление с вычисляемым столбцом.

☐ Создано агрегированное представление.

☐ Создано аналитическое представление.

---

## Управление представлениями

☐ Использован `CREATE VIEW`.

☐ Использован `ALTER VIEW`.

☐ Использован `CREATE OR ALTER VIEW`.

☐ Использован `DROP VIEW`.

☐ Просмотрен список через `sys.views`.

☐ Получено определение представления.

☐ Проверены зависимости представления.

---

## Изменение данных

☐ Проверен `INSERT` через простое представление.

☐ Проверен `UPDATE` через простое представление.

☐ Проверена попытка изменения через сложное представление.

☐ Использован `WITH CHECK OPTION`.

☐ Проверено его влияние на изменение данных.

---

## SCHEMABINDING

☐ Создано представление с `SCHEMABINDING`.

☐ Выполнена попытка изменить зависимую структуру таблицы.

☐ Зафиксирован результат.

---

## Производительность

☐ Выполнен прямой запрос к таблицам.

☐ Выполнен эквивалентный запрос через представление.

☐ Включен `STATISTICS IO`.

☐ Включен `STATISTICS TIME`.

☐ Получены фактические планы выполнения.

☐ Сравнены Logical Reads.

☐ Сравнены планы.

☐ Проверено влияние индекса базовой таблицы.

☐ Проверено агрегированное представление.

---

## Проектирование

☐ Не используется необоснованный `SELECT *`.

☐ Нет лишней вложенности представлений.

☐ Каждое представление имеет конкретное назначение.

☐ Названия отражают назначение представления.

☐ Созданы минимум три собственных представления.

☐ Сформирована итоговая таблица объектов.

# Советы по улучшению работы

Не создавайте представление только ради того, чтобы скрыть один короткий `SELECT`. У представления должно быть понятное назначение: упрощение сложного запроса, формирование стабильного интерфейса доступа к данным, ограничение доступных столбцов или подготовка данных для отчетности.

Не следует считать представление инструментом автоматической оптимизации. Если запрос внутри представления выполняется плохо, его перенос в `VIEW` обычно не устранит проблему.

Особенно внимательно анализируйте представления с большим количеством `JOIN`. Чем больше таблиц участвует в запросе, тем важнее корректные внешние ключи, индексы и условия соединения.

Не создавайте одно универсальное представление с десятками таблиц и сотнями столбцов для всей системы. Такие объекты быстро становятся трудными для сопровождения.

Старайтесь проектировать представления под конкретные сценарии использования.

Например:

```text
vw_ActiveCustomers
```

понятно предназначено для работы с активными клиентами.

А название:

```text
vw_Data
```

ничего не сообщает о назначении объекта.

При разработке интерфейсов для приложений представления могут быть полезны как стабильный слой между приложением и внутренней структурой базы данных. Таблицы внутри системы могут изменяться, а контракт представления может оставаться более стабильным.

Если через представление предполагается изменять данные, обязательно проверяйте это отдельными `INSERT`, `UPDATE` и `DELETE`. Не любое представление является обновляемым.

Для фильтрованных изменяемых представлений рассмотрите использование:

```sql
WITH CHECK OPTION
```

чтобы через представление нельзя было создать или изменить строку таким образом, что она перестанет соответствовать логике самого представления.

При работе с производительностью всегда анализируйте исходные таблицы. Индекс на столбце `CustomerId` базовой таблицы может существенно ускорить запрос через представление, даже если само представление не менялось.

Итоговая работа считается качественно выполненной, если представления имеют понятное назначение, не дублируют бессмысленно друг друга, корректно работают с исходными таблицами и проверены не только функционально, но и через фактические планы выполнения и статистику SQL Server.
