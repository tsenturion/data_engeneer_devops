# Практическое задание

# Проектирование и реализация хранимых процедур в MS SQL Server

## Цель

Спроектировать и реализовать набор хранимых процедур в MS SQL Server для выполнения типовых операций с данными, передачи входных параметров, возврата результатов, проверки передаваемых значений и управления контекстом выполнения.

В ходе практической работы необходимо создать базу данных с несколькими связанными таблицами, реализовать процедуры для чтения, добавления, изменения и удаления данных, разработать параметризованные процедуры, использовать выходные параметры и возвращаемые значения, выполнить обработку ошибок и проверить выполнение процедур в различных контекстах безопасности.

Основная задача работы — использовать хранимую процедуру не просто как сохраненный `SELECT`, а как серверный программный объект, который инкапсулирует определенную операцию и предоставляет контролируемый интерфейс работы с базой данных.

# Задание

Необходимо создать небольшую систему обработки заявок клиентов.

В базе данных должны храниться:

- клиенты;
- сотрудники;
- категории заявок;
- заявки;
- история изменения статусов заявок.

На основе этих таблиц необходимо разработать хранимые процедуры различного назначения.

Минимально должны быть реализованы:

- процедура без параметров;
- процедура с одним входным параметром;
- процедура с несколькими параметрами;
- процедура с необязательными параметрами;
- процедура добавления данных;
- процедура изменения данных;
- процедура удаления данных;
- процедура с выходным параметром;
- процедура с `RETURN`;
- процедура с проверкой входных значений;
- процедура с транзакцией;
- процедура с обработкой ошибок;
- процедура с заданным контекстом исполнения.

---

# 1. Создание базы данных

Создать отдельную базу данных:

```sql
CREATE DATABASE ProceduresPracticeDB;
GO

USE ProceduresPracticeDB;
GO
```

Создать отдельную схему:

```sql
CREATE SCHEMA Support;
GO
```

Все основные объекты практической работы разместить в схеме `Support`.

---

# 2. Создание таблицы клиентов

Создать таблицу клиентов:

```sql
CREATE TABLE Support.Customers
(
    CustomerId INT IDENTITY(1,1)
        CONSTRAINT PK_Customers PRIMARY KEY,

    FullName NVARCHAR(200) NOT NULL,

    Email NVARCHAR(200) NOT NULL,

    Phone NVARCHAR(30) NULL,

    City NVARCHAR(100) NOT NULL,

    RegistrationDate DATETIME2 NOT NULL
        CONSTRAINT DF_Customers_RegistrationDate
        DEFAULT SYSDATETIME(),

    IsActive BIT NOT NULL
        CONSTRAINT DF_Customers_IsActive
        DEFAULT 1,

    CONSTRAINT UQ_Customers_Email
        UNIQUE (Email)
);
```

---

# 3. Создание таблицы сотрудников

```sql
CREATE TABLE Support.Employees
(
    EmployeeId INT IDENTITY(1,1)
        CONSTRAINT PK_Employees PRIMARY KEY,

    FullName NVARCHAR(200) NOT NULL,

    Position NVARCHAR(100) NOT NULL,

    IsActive BIT NOT NULL
        CONSTRAINT DF_Employees_IsActive
        DEFAULT 1
);
```

---

# 4. Создание таблицы категорий

```sql
CREATE TABLE Support.RequestCategories
(
    CategoryId INT IDENTITY(1,1)
        CONSTRAINT PK_RequestCategories PRIMARY KEY,

    CategoryName NVARCHAR(100) NOT NULL,

    IsActive BIT NOT NULL
        CONSTRAINT DF_RequestCategories_IsActive
        DEFAULT 1,

    CONSTRAINT UQ_RequestCategories_CategoryName
        UNIQUE (CategoryName)
);
```

---

# 5. Создание таблицы заявок

```sql
CREATE TABLE Support.Requests
(
    RequestId BIGINT IDENTITY(1,1)
        CONSTRAINT PK_Requests PRIMARY KEY,

    CustomerId INT NOT NULL,

    CategoryId INT NOT NULL,

    EmployeeId INT NULL,

    Title NVARCHAR(200) NOT NULL,

    Description NVARCHAR(1000) NULL,

    Priority VARCHAR(20) NOT NULL,

    Status VARCHAR(30) NOT NULL,

    CreatedAt DATETIME2 NOT NULL
        CONSTRAINT DF_Requests_CreatedAt
        DEFAULT SYSDATETIME(),

    ClosedAt DATETIME2 NULL,

    CONSTRAINT FK_Requests_Customers
        FOREIGN KEY (CustomerId)
        REFERENCES Support.Customers(CustomerId),

    CONSTRAINT FK_Requests_Categories
        FOREIGN KEY (CategoryId)
        REFERENCES Support.RequestCategories(CategoryId),

    CONSTRAINT FK_Requests_Employees
        FOREIGN KEY (EmployeeId)
        REFERENCES Support.Employees(EmployeeId),

    CONSTRAINT CK_Requests_Priority
        CHECK
        (
            Priority IN
            (
                'Low',
                'Normal',
                'High',
                'Critical'
            )
        ),

    CONSTRAINT CK_Requests_Status
        CHECK
        (
            Status IN
            (
                'New',
                'InProgress',
                'Resolved',
                'Closed',
                'Cancelled'
            )
        )
);
```

---

# 6. Создание таблицы истории

```sql
CREATE TABLE Support.RequestStatusHistory
(
    HistoryId BIGINT IDENTITY(1,1)
        CONSTRAINT PK_RequestStatusHistory PRIMARY KEY,

    RequestId BIGINT NOT NULL,

    OldStatus VARCHAR(30) NULL,

    NewStatus VARCHAR(30) NOT NULL,

    ChangedAt DATETIME2 NOT NULL
        CONSTRAINT DF_RequestStatusHistory_ChangedAt
        DEFAULT SYSDATETIME(),

    ChangedByEmployeeId INT NULL,

    CONSTRAINT FK_RequestStatusHistory_Requests
        FOREIGN KEY (RequestId)
        REFERENCES Support.Requests(RequestId),

    CONSTRAINT FK_RequestStatusHistory_Employees
        FOREIGN KEY (ChangedByEmployeeId)
        REFERENCES Support.Employees(EmployeeId)
);
```

---

# 7. Заполнение базы тестовыми данными

Добавить минимум:

```text
Customers — 20 записей
Employees — 5 записей
RequestCategories — 5 записей
Requests — 50 записей
```

Заявки должны иметь разные:

```text
статусы;
приоритеты;
категории;
клиентов;
сотрудников;
даты создания.
```

Часть заявок должна оставаться без назначенного сотрудника.

---

# 8. Создание простой процедуры без параметров

Создать процедуру, которая возвращает список всех открытых заявок.

```sql
CREATE PROCEDURE Support.usp_GetOpenRequests
AS
BEGIN
    SET NOCOUNT ON;

    SELECT
        RequestId,
        CustomerId,
        CategoryId,
        EmployeeId,
        Title,
        Priority,
        Status,
        CreatedAt
    FROM Support.Requests
    WHERE Status IN
    (
        'New',
        'InProgress'
    )
    ORDER BY CreatedAt;
END;
GO
```

Выполнить:

```sql
EXEC Support.usp_GetOpenRequests;
```

Проверить результат.

---

# 9. Исследование SET NOCOUNT ON

В первой процедуре оставить:

```sql
SET NOCOUNT ON;
```

Создать временную аналогичную процедуру без этой инструкции.

Сравнить вывод при выполнении операций.

Определить назначение:

```sql
SET NOCOUNT ON;
```

После эксперимента временную процедуру удалить.

---

# 10. Создание процедуры с одним параметром

Создать процедуру получения заявки по идентификатору:

```sql
CREATE PROCEDURE Support.usp_GetRequestById
    @RequestId BIGINT
AS
BEGIN
    SET NOCOUNT ON;

    SELECT
        r.RequestId,
        r.Title,
        r.Description,
        r.Priority,
        r.Status,
        r.CreatedAt,
        r.ClosedAt
    FROM Support.Requests AS r
    WHERE r.RequestId = @RequestId;
END;
GO
```

Выполнить:

```sql
EXEC Support.usp_GetRequestById
    @RequestId = 10;
```

Проверить существующий и несуществующий идентификатор.

---

# 11. Создание процедуры с несколькими параметрами

Создать процедуру поиска заявок по статусу и приоритету:

```sql
CREATE PROCEDURE Support.usp_GetRequestsByStatusAndPriority
    @Status VARCHAR(30),
    @Priority VARCHAR(20)
AS
BEGIN
    SET NOCOUNT ON;

    SELECT
        RequestId,
        Title,
        Status,
        Priority,
        EmployeeId,
        CreatedAt
    FROM Support.Requests
    WHERE Status = @Status
      AND Priority = @Priority
    ORDER BY CreatedAt DESC;
END;
GO
```

Выполнить несколько вариантов:

```sql
EXEC Support.usp_GetRequestsByStatusAndPriority
    @Status = 'New',
    @Priority = 'High';
```

```sql
EXEC Support.usp_GetRequestsByStatusAndPriority
    @Status = 'InProgress',
    @Priority = 'Critical';
```

---

# 12. Создание процедуры с необязательными параметрами

Создать процедуру поиска заявок с несколькими фильтрами.

Параметры должны иметь значения по умолчанию:

```sql
CREATE PROCEDURE Support.usp_SearchRequests
    @CustomerId INT = NULL,
    @EmployeeId INT = NULL,
    @Status VARCHAR(30) = NULL,
    @Priority VARCHAR(20) = NULL
AS
BEGIN
    SET NOCOUNT ON;

    SELECT
        RequestId,
        CustomerId,
        EmployeeId,
        Title,
        Status,
        Priority,
        CreatedAt
    FROM Support.Requests
    WHERE
        (@CustomerId IS NULL OR CustomerId = @CustomerId)
        AND
        (@EmployeeId IS NULL OR EmployeeId = @EmployeeId)
        AND
        (@Status IS NULL OR Status = @Status)
        AND
        (@Priority IS NULL OR Priority = @Priority)
    ORDER BY CreatedAt DESC;
END;
GO
```

Проверить разные способы выполнения.

Без параметров:

```sql
EXEC Support.usp_SearchRequests;
```

По статусу:

```sql
EXEC Support.usp_SearchRequests
    @Status = 'New';
```

По сотруднику и приоритету:

```sql
EXEC Support.usp_SearchRequests
    @EmployeeId = 2,
    @Priority = 'High';
```

Проанализировать преимущество именованных параметров.

---

# 13. Создание процедуры добавления клиента

Создать процедуру:

```sql
CREATE PROCEDURE Support.usp_CreateCustomer
    @FullName NVARCHAR(200),
    @Email NVARCHAR(200),
    @Phone NVARCHAR(30) = NULL,
    @City NVARCHAR(100)
AS
BEGIN
    SET NOCOUNT ON;

    INSERT INTO Support.Customers
    (
        FullName,
        Email,
        Phone,
        City
    )
    VALUES
    (
        @FullName,
        @Email,
        @Phone,
        @City
    );
END;
GO
```

Выполнить:

```sql
EXEC Support.usp_CreateCustomer
    @FullName = N'Иван Сергеев',
    @Email = N'ivan.sergeev@example.com',
    @Phone = N'+7 900 100-20-30',
    @City = N'Москва';
```

Проверить появление клиента в таблице.

---

# 14. Возврат идентификатора созданной записи

Изменить процедуру создания клиента таким образом, чтобы она возвращала созданный `CustomerId`.

Для получения значения использовать:

```sql
SCOPE_IDENTITY()
```

Пример:

```sql
CREATE OR ALTER PROCEDURE Support.usp_CreateCustomer
    @FullName NVARCHAR(200),
    @Email NVARCHAR(200),
    @Phone NVARCHAR(30) = NULL,
    @City NVARCHAR(100),
    @NewCustomerId INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    INSERT INTO Support.Customers
    (
        FullName,
        Email,
        Phone,
        City
    )
    VALUES
    (
        @FullName,
        @Email,
        @Phone,
        @City
    );

    SET @NewCustomerId =
        CONVERT(INT, SCOPE_IDENTITY());
END;
GO
```

Проверить работу выходного параметра:

```sql
DECLARE @CustomerId INT;

EXEC Support.usp_CreateCustomer
    @FullName = N'Анна Белова',
    @Email = N'anna.belova@example.com',
    @Phone = NULL,
    @City = N'Казань',
    @NewCustomerId = @CustomerId OUTPUT;

SELECT @CustomerId AS NewCustomerId;
```

---

# 15. Создание процедуры добавления заявки

Разработать процедуру:

```sql
Support.usp_CreateRequest
```

Она должна принимать:

```text
CustomerId
CategoryId
Title
Description
Priority
```

Статус новой заявки должен автоматически устанавливаться:

```text
New
```

Дата создания должна задаваться средствами базы данных.

Процедура должна возвращать идентификатор созданной заявки через `OUTPUT`.

---

# 16. Проверка существования связанных данных

Улучшить процедуру создания заявки.

Перед выполнением `INSERT` проверить существование клиента:

```sql
IF NOT EXISTS
(
    SELECT 1
    FROM Support.Customers
    WHERE CustomerId = @CustomerId
      AND IsActive = 1
)
```

Если клиент отсутствует или неактивен, операция не должна выполняться.

Аналогичную проверку сделать для категории заявки.

Для передачи ошибки использовать:

```sql
THROW
```

Например:

```sql
THROW 50001, N'Клиент не найден или неактивен.', 1;
```

---

# 17. Проверка допустимости параметров

В процедуре создания заявки проверить значение приоритета.

Разрешенные значения:

```text
Low
Normal
High
Critical
```

Перед выполнением `INSERT` реализовать проверку.

Например:

```sql
IF @Priority NOT IN
(
    'Low',
    'Normal',
    'High',
    'Critical'
)
BEGIN
    THROW 50002, N'Недопустимый приоритет заявки.', 1;
END;
```

Проверить процедуру с корректным и некорректным значением.

---

# 18. Создание процедуры изменения данных

Создать процедуру изменения контактных данных клиента:

```sql
Support.usp_UpdateCustomerContacts
```

Параметры:

```text
CustomerId
Email
Phone
City
```

Процедура должна:

```text
проверить наличие клиента;
изменить данные;
не изменять FullName;
не изменять CustomerId;
```

После выполнения проверить результат.

---

# 19. Проверка количества измененных строк

В процедуре изменения клиента использовать:

```sql
@@ROWCOUNT
```

Например:

```sql
UPDATE Support.Customers
SET
    Email = @Email,
    Phone = @Phone,
    City = @City
WHERE CustomerId = @CustomerId;

IF @@ROWCOUNT = 0
BEGIN
    THROW 50003, N'Клиент не найден.', 1;
END;
```

Проверить выполнение для существующего и отсутствующего клиента.

---

# 20. Создание процедуры назначения сотрудника

Создать:

```sql
Support.usp_AssignEmployee
```

Параметры:

```text
RequestId
EmployeeId
```

Перед назначением проверить:

```text
существует ли заявка;
существует ли сотрудник;
активен ли сотрудник;
не имеет ли заявка статус Closed или Cancelled.
```

Если условия выполнены, назначить сотрудника:

```sql
UPDATE Support.Requests
SET
    EmployeeId = @EmployeeId,
    Status = 'InProgress'
WHERE RequestId = @RequestId;
```

---

# 21. Создание процедуры изменения статуса заявки

Создать:

```sql
Support.usp_ChangeRequestStatus
```

Параметры:

```text
RequestId
NewStatus
EmployeeId
```

Процедура должна:

```text
проверять наличие заявки;
получать предыдущий статус;
изменять текущий статус;
добавлять запись в RequestStatusHistory.
```

Если новый статус:

```text
Closed
```

в столбец:

```text
ClosedAt
```

необходимо записать текущую дату и время.

---

# 22. Использование локальных переменных внутри процедуры

В процедуре изменения статуса использовать переменную:

```sql
DECLARE @OldStatus VARCHAR(30);
```

Получить текущее значение:

```sql
SELECT
    @OldStatus = Status
FROM Support.Requests
WHERE RequestId = @RequestId;
```

После изменения добавить в историю:

```sql
INSERT INTO Support.RequestStatusHistory
(
    RequestId,
    OldStatus,
    NewStatus,
    ChangedByEmployeeId
)
VALUES
(
    @RequestId,
    @OldStatus,
    @NewStatus,
    @EmployeeId
);
```

---

# 23. Использование транзакции в процедуре

Изменение статуса заявки и добавление записи в историю являются одной логической операцией.

Реализовать их внутри транзакции:

```sql
BEGIN TRANSACTION;
```

Если обе операции успешны:

```sql
COMMIT TRANSACTION;
```

При ошибке:

```sql
ROLLBACK TRANSACTION;
```

Необходимо добиться того, чтобы не возникала ситуация, когда статус изменен, но история изменения не записана.

---

# 24. Использование TRY...CATCH

Переработать процедуру изменения статуса:

```sql
BEGIN TRY

    BEGIN TRANSACTION;

    -- Изменение заявки.

    -- Добавление истории.

    COMMIT TRANSACTION;

END TRY
BEGIN CATCH

    IF @@TRANCOUNT > 0
        ROLLBACK TRANSACTION;

    THROW;

END CATCH;
```

Проверить нормальное выполнение.

После этого намеренно создать ситуацию, вызывающую ошибку, и убедиться, что транзакция откатывается.

---

# 25. Создание процедуры удаления данных

Создать процедуру:

```sql
Support.usp_DeleteRequest
```

Она должна удалять заявку только в том случае, если ее статус:

```text
Cancelled
```

или соответствует другой логике, самостоятельно выбранной для работы.

Если заявка находится в рабочем статусе, удаление должно блокироваться.

Необходимо учитывать наличие строк в:

```text
Support.RequestStatusHistory
```

и ссылочную целостность.

Выбрать и реализовать корректную стратегию удаления.

---

# 26. Использование логического удаления

Создать альтернативный подход, при котором клиент физически не удаляется.

Создать процедуру:

```sql
Support.usp_DeactivateCustomer
```

Она должна выполнять:

```sql
UPDATE Support.Customers
SET IsActive = 0
WHERE CustomerId = @CustomerId;
```

Проанализировать различие между:

```text
физическим удалением;
логическим отключением записи.
```

Для рабочей системы выбрать более подходящий вариант для клиентов.

---

# 27. Создание процедуры с OUTPUT-параметром

Создать процедуру подсчета открытых заявок клиента:

```sql
CREATE PROCEDURE Support.usp_GetCustomerOpenRequestCount
    @CustomerId INT,
    @OpenRequestCount INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    SELECT
        @OpenRequestCount = COUNT(*)
    FROM Support.Requests
    WHERE CustomerId = @CustomerId
      AND Status IN
      (
          'New',
          'InProgress'
      );
END;
GO
```

Проверить:

```sql
DECLARE @Count INT;

EXEC Support.usp_GetCustomerOpenRequestCount
    @CustomerId = 1,
    @OpenRequestCount = @Count OUTPUT;

SELECT @Count AS OpenRequests;
```

---

# 28. Создание процедуры с несколькими OUTPUT-параметрами

Создать процедуру:

```sql
Support.usp_GetRequestStatistics
```

Она должна возвращать через выходные параметры:

```text
общее количество заявок;
количество открытых;
количество закрытых;
количество критических.
```

Например:

```sql
@TotalCount INT OUTPUT,
@OpenCount INT OUTPUT,
@ClosedCount INT OUTPUT,
@CriticalCount INT OUTPUT
```

После выполнения вывести все полученные значения одним `SELECT`.

---

# 29. Использование RETURN

Создать процедуру, которая проверяет существование клиента.

Например:

```sql
CREATE PROCEDURE Support.usp_CheckCustomer
    @CustomerId INT
AS
BEGIN
    SET NOCOUNT ON;

    IF EXISTS
    (
        SELECT 1
        FROM Support.Customers
        WHERE CustomerId = @CustomerId
    )
        RETURN 0;

    RETURN 1;
END;
GO
```

Выполнить:

```sql
DECLARE @Result INT;

EXEC @Result = Support.usp_CheckCustomer
    @CustomerId = 10;

SELECT @Result AS ReturnCode;
```

Установить собственное соглашение:

```text
0 — успешно;
1 — клиент не найден.
```

---

# 30. Сравнение RETURN и OUTPUT

На основании созданных процедур сравнить два механизма.

`RETURN` использовать для небольшого целочисленного кода результата выполнения.

`OUTPUT` использовать для возврата значений, вычисленных процедурой.

Не использовать `RETURN` в качестве замены полноценного результирующего набора данных.

---

# 31. Возврат результирующего набора

Создать процедуру:

```sql
Support.usp_GetCustomerRequests
```

которая принимает:

```text
CustomerId
```

и возвращает через обычный `SELECT` все заявки клиента.

Результат должен содержать:

```text
RequestId
Title
Priority
Status
CreatedAt
EmployeeId
```

Сравнить такой способ возврата данных с `OUTPUT`-параметром.

---

# 32. Использование нескольких результирующих наборов

Создать экспериментальную процедуру:

```sql
Support.usp_GetCustomerDashboard
```

Она должна вернуть два результирующих набора.

Первый:

```text
основные сведения о клиенте.
```

Второй:

```text
список его заявок.
```

Проверить выполнение процедуры в DBeaver.

Определить, как клиентская программа отображает несколько результатов.

---

# 33. Создание процедуры отчетности

Создать:

```sql
Support.usp_GetRequestReport
```

Параметры:

```text
DateFrom
DateTo
Status
EmployeeId
```

`Status` и `EmployeeId` сделать необязательными.

Процедура должна возвращать заявки за указанный период.

Обязательные параметры:

```sql
@DateFrom DATE,
@DateTo DATE
```

Необходимо проверить:

```text
DateFrom не должна быть больше DateTo.
```

При нарушении условия использовать `THROW`.

---

# 34. Проверка работы параметров дат

Выполнить отчет:

```sql
EXEC Support.usp_GetRequestReport
    @DateFrom = '2026-01-01',
    @DateTo = '2026-12-31';
```

Затем выполнить:

```sql
EXEC Support.usp_GetRequestReport
    @DateFrom = '2026-01-01',
    @DateTo = '2026-12-31',
    @Status = 'Closed';
```

Затем добавить фильтр сотрудника.

Проверить корректность всех вариантов.

---

# 35. Изменение существующей процедуры

Использовать:

```sql
ALTER PROCEDURE
```

для изменения одной из созданных процедур.

Например, добавить в результат:

```text
CategoryId
```

или другой столбец.

После изменения проверить выполнение.

---

# 36. Использование CREATE OR ALTER PROCEDURE

Для одной из процедур применить:

```sql
CREATE OR ALTER PROCEDURE
```

Например:

```sql
CREATE OR ALTER PROCEDURE Support.usp_GetOpenRequests
AS
BEGIN
    SET NOCOUNT ON;

    SELECT
        RequestId,
        CustomerId,
        Title,
        Priority,
        Status,
        CreatedAt
    FROM Support.Requests
    WHERE Status IN
    (
        'New',
        'InProgress'
    )
    ORDER BY Priority, CreatedAt;
END;
GO
```

Определить преимущество `CREATE OR ALTER` при разработке и повторном запуске скриптов.

---

# 37. Получение списка процедур

Получить информацию о созданных объектах:

```sql
SELECT
    SCHEMA_NAME(schema_id) AS schemaname,
    name AS procedurename,
    create_date,
    modify_date
FROM sys.procedures
WHERE schema_id = SCHEMA_ID('Support')
ORDER BY name;
```

---

# 38. Просмотр определения процедуры

Получить определение:

```sql
SELECT
    OBJECT_DEFINITION(
        OBJECT_ID('Support.usp_CreateRequest')
    );
```

Также проверить:

```sql
EXEC sp_helptext 'Support.usp_CreateRequest';
```

---

# 39. Получение информации о параметрах

Использовать:

```sql
SELECT
    p.name AS parametername,
    TYPE_NAME(p.user_type_id) AS datatype,
    p.max_length,
    p.is_output
FROM sys.parameters AS p
WHERE
    p.object_id
    = OBJECT_ID('Support.usp_CreateRequest');
```

Проверить параметры нескольких процедур.

---

# 40. Управление контекстом исполнения

Создать отдельную процедуру для исследования контекста:

```sql
CREATE PROCEDURE Support.usp_ShowExecutionContext
AS
BEGIN
    SET NOCOUNT ON;

    SELECT
        ORIGINAL_LOGIN() AS OriginalLogin,
        SUSER_SNAME() AS CurrentLogin,
        USER_NAME() AS CurrentDatabaseUser;
END;
GO
```

Выполнить:

```sql
EXEC Support.usp_ShowExecutionContext;
```

Зафиксировать результат.

---

# 41. Создание тестового пользователя базы данных

Для исследования контекста исполнения создать пользователя базы данных в соответствии с доступной конфигурацией лабораторного SQL Server.

Например, при допустимости пользователя без логина:

```sql
CREATE USER ProcedureTestUser
WITHOUT LOGIN;
```

Не выдавать этому пользователю прямой доступ ко всем таблицам.

---

# 42. Проверка EXECUTE AS

В рамках тестирования выполнить:

```sql
EXECUTE AS USER = 'ProcedureTestUser';
```

Проверить текущий контекст:

```sql
SELECT USER_NAME() AS CURRENTDATABASEUSER;
```

Попытаться обратиться напрямую к таблице:

```sql
SELECT *
FROM Support.Customers;
```

Зафиксировать результат.

После эксперимента вернуть исходный контекст:

```sql
REVERT;
```

Обязательно проверить:

```sql
SELECT USER_NAME() AS CURRENTDATABASEUSER;
```

---

# 43. Создание процедуры с EXECUTE AS

Создать процедуру с явно указанным контекстом исполнения.

Например:

```sql
CREATE PROCEDURE Support.usp_GetCustomerDirectory
WITH EXECUTE AS OWNER
AS
BEGIN
    SET NOCOUNT ON;

    SELECT
        CustomerId,
        FullName,
        City,
        IsActive
    FROM Support.Customers;
END;
GO
```

Проверить ее выполнение в тестовом контексте.

Сравнить:

```text
прямой SELECT к таблице;
выполнение процедуры.
```

Зафиксировать результат.

---

# 44. Исследование CALLER и OWNER

Для экспериментальной процедуры проверить варианты контекста:

```sql
WITH EXECUTE AS CALLER
```

и:

```sql
WITH EXECUTE AS OWNER
```

Сравнить поведение при выполнении от имени тестового пользователя.

Определить, чьи разрешения используются в каждом случае.

После эксперимента оставить только необходимый вариант.

---

# 45. Проверка ORIGINAL_LOGIN, SUSER_SNAME и USER_NAME

Внутри экспериментальной процедуры вывести:

```sql
SELECT
    ORIGINAL_LOGIN() AS ORIGINALLOGIN,
    SUSER_SNAME() AS SERVERIDENTITY,
    USER_NAME() AS DATABASEIDENTITY;
```

Выполнить процедуру:

```text
в обычном контексте;
после EXECUTE AS;
через процедуру с EXECUTE AS OWNER.
```

Сравнить значения.

---

# 46. Проверка прав на EXECUTE

Если конфигурация лабораторной среды позволяет, предоставить тестовому пользователю право выполнения конкретной процедуры:

```sql
GRANT EXECUTE
ON Support.Usp_getcustomerdirectory
TO Proceduretestuser;
```

При этом не выдавать пользователю общий `SELECT` на таблицу `Support.Customers`.

Проверить, может ли пользователь использовать разрешенную процедуру без прямого доступа к исходной таблице.

---

# 47. Создание процедуры безопасного изменения данных

Создать процедуру:

```sql
Support.usp_CloseRequest
```

которая должна закрывать заявку только при соблюдении правил.

Например:

```text
заявка существует;
она не находится в статусе Cancelled;
назначен сотрудник;
текущий статус не Closed.
```

Процедура должна:

```text
обновить Status;
записать ClosedAt;
создать строку истории;
выполнить все действия в одной транзакции.
```

Для ошибок использовать `THROW`.

---

# 48. Проверка XACT_STATE

В обработчике ошибок процедуры с транзакцией дополнительно исследовать:

```sql
XACT_STATE()
```

Например:

```sql
BEGIN CATCH

    IF XACT_STATE() <> 0
        ROLLBACK TRANSACTION;

    THROW;

END CATCH;
```

Проверить состояние транзакции в корректном и ошибочном сценарии.

---

# 49. Проверка XACT_ABORT

Для одной процедуры с транзакцией установить:

```sql
SET XACT_ABORT ON;
```

Например:

```sql
CREATE OR ALTER PROCEDURE Support.usp_CloseRequest
    @RequestId BIGINT,
    @EmployeeId INT
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;

    -- Основная логика.
END;
GO
```

Исследовать назначение этой настройки при выполнении транзакционных процедур.

---

# 50. Проверка повторного выполнения

Для процедур добавления данных определить, что произойдет при повторном выполнении с теми же параметрами.

Особенно проверить:

```text
повторяющийся Email клиента;
повторное закрытие заявки;
повторное назначение сотрудника.
```

Необходимо решить, какие операции должны:

```text
завершаться ошибкой;
ничего не изменять;
возвращать существующий результат.
```

---

# 51. Проверка производительности параметризованной процедуры

Выбрать одну процедуру выборки, например:

```text
Support.usp_SearchRequests
```

Включить:

```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
```

Выполнить ее с разными параметрами.

Например:

```sql
EXEC Support.usp_SearchRequests
    @Status = 'New';
```

и:

```sql
EXEC Support.usp_SearchRequests
    @CustomerId = 1;
```

Зафиксировать планы выполнения и статистику.

Задача этого этапа — увидеть, что одна параметризованная процедура может использоваться для разных вариантов поиска, но универсальность процедуры также может влиять на сложность оптимизации.

---

# 52. Просмотр кешированных планов процедур

Найти планы созданных процедур через системные представления.

Например, исследовать:

```sql
sys.dm_exec_procedure_stats
```

Создать запрос, который выводит:

```text
имя процедуры;
количество выполнений;
последнее время выполнения;
суммарное CPU;
суммарное время выполнения.
```

Один из возможных вариантов:

```sql
SELECT
    OBJECT_SCHEMA_NAME(ps.object_id, ps.database_id)
        AS schemaname,

    OBJECT_NAME(ps.object_id, ps.database_id)
        AS procedurename,

    ps.execution_count,
    ps.total_worker_time,
    ps.total_elapsed_time,
    ps.last_execution_time

FROM sys.dm_exec_procedure_stats AS ps
WHERE ps.database_id = DB_ID()
ORDER BY ps.total_elapsed_time DESC;
```

Выполнить несколько процедур несколько раз и повторить запрос.

---

# 53. Удаление экспериментальной процедуры

Создать временную процедуру:

```sql
CREATE PROCEDURE Support.usp_TestProcedure
AS
BEGIN
    SELECT N'Тестовая процедура' AS Message;
END;
GO
```

Проверить ее выполнение.

После этого удалить:

```sql
DROP PROCEDURE Support.usp_TestProcedure;
GO
```

Проверить:

```sql
SELECT *
FROM sys.procedures
WHERE name = 'usp_TestProcedure';
```

---

# 54. Самостоятельная разработка процедуры

Самостоятельно разработать одну комплексную процедуру.

Выбрать один вариант:

```text
переназначение заявки другому сотруднику;
повторное открытие закрытой заявки;
массовое закрытие старых заявок;
получение статистики сотрудника;
получение статистики клиента;
создание клиента и первой заявки одной операцией;
смена приоритета заявки с записью истории.
```

Процедура должна содержать минимум:

```text
два входных параметра;
проверку входных данных;
минимум одну условную конструкцию;
работу минимум с двумя таблицами;
обработку ошибки;
осмысленный результат выполнения.
```

Если операция изменяет несколько связанных таблиц, использовать транзакцию.

---

# 55. Подготовка итоговой таблицы процедур

Сформировать таблицу:

| Процедура                | Назначение                | Входные параметры                | OUTPUT     | RETURN   | Изменяет данные | Транзакция |
| ------------------------ | ------------------------- | -------------------------------- | ---------- | -------- | --------------- | ---------- |
| usp_GetOpenRequests      | Получение открытых заявок | Нет                              | Нет        | Нет      | Нет             | Нет        |
| usp_GetRequestById       | Получение заявки          | RequestId                        | Нет        | Нет      | Нет             | Нет        |
| usp_CreateCustomer       | Создание клиента          | Данные клиента                   | CustomerId | Нет      | Да              | Нет        |
| usp_CreateRequest        | Создание заявки           | Данные заявки                    | RequestId  | Возможен | Да              | По решению |
| usp_ChangeRequestStatus  | Изменение статуса         | RequestId, NewStatus, EmployeeId | Нет        | Возможен | Да              | Да         |
| usp_GetRequestStatistics | Статистика                | —                                | Несколько  | Нет      | Нет             | Нет        |

Добавить в таблицу все самостоятельно созданные процедуры.

# Подсказки по ключевым частям

Хранимая процедура должна рассматриваться как программный интерфейс базы данных. Вместо того чтобы приложение самостоятельно выполнять несколько запросов:

```sql
SELECT ...
UPDATE ...
INSERT ...
```

часть логики можно оформить как одну серверную операцию:

```sql
EXEC Support.usp_ChangeRequestStatus
    @RequestId = 100,
    @NewStatus = 'Closed',
    @EmployeeId = 3;
```

Внутри процедуры можно централизованно выполнять проверки, изменять несколько таблиц и обрабатывать ошибки.

Параметры позволяют отказаться от создания отдельных процедур для каждого конкретного значения.

Не требуется создавать:

```text
usp_GetNewRequests
usp_GetClosedRequests
usp_GetCriticalRequests
```

если задача рационально решается параметризованной процедурой.

Например:

```sql
EXEC Support.usp_GetRequestsByStatusAndPriority
    @Status = 'Closed',
    @Priority = 'Critical';
```

При этом не следует создавать одну универсальную процедуру на десятки необязательных параметров без необходимости. Чем больше вариантов поведения скрыто внутри одной процедуры, тем сложнее ее сопровождение и анализ производительности.

Входной параметр передает значение в процедуру:

```sql
@RequestId BIGINT
```

Выходной параметр позволяет процедуре передать вычисленное значение вызывающему коду:

```sql
@NewRequestId BIGINT OUTPUT
```

Необходимо помнить, что слово `OUTPUT` указывается и при объявлении параметра процедуры, и при ее вызове:

```sql
DECLARE @Id BIGINT;

EXEC Support.usp_CreateRequest
    ...,
    @NewRequestId = @Id OUTPUT;
```

`RETURN` имеет другое назначение. Его удобно использовать для небольшого целочисленного кода результата:

```text
0 — операция успешна;
1 — объект не найден;
2 — операция запрещена.
```

Для возврата таблицы данных использовать обычный:

```sql
SELECT
```

Для возврата конкретного вычисленного значения можно использовать:

```text
OUTPUT-параметр.
```

Для кода состояния можно использовать:

```text
RETURN.
```

Перед изменением данных необходимо проверять бизнес-условия. Внешний ключ способен защитить ссылочную целостность, но он не знает прикладной логики.

Например, внешний ключ подтвердит, что сотрудник существует, но не проверит, что:

```text
сотрудник активен;
заявка еще не закрыта;
сотруднику разрешено работать с этой категорией.
```

Такую проверку можно выполнять внутри процедуры.

Если одна логическая операция изменяет несколько таблиц, необходимо рассматривать транзакцию.

Например, смена статуса:

```text
изменяет Requests;
добавляет RequestStatusHistory.
```

Эти изменения должны выполняться совместно. Состояние, когда обновление основной таблицы прошло успешно, а история не сохранилась, является нарушением согласованности.

Поэтому используется:

```sql
BEGIN TRANSACTION;
```

а затем:

```sql
COMMIT;
```

или при ошибке:

```sql
ROLLBACK;
```

Для процедур, содержащих транзакции, рекомендуется использовать:

```sql
TRY
CATCH
THROW
```

Например:

```sql
BEGIN TRY
    BEGIN TRANSACTION;

    -- Изменения данных.

    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF XACT_STATE() <> 0
        ROLLBACK TRANSACTION;

    THROW;
END CATCH;
```

`THROW` позволяет передать ошибку вызывающей стороне и не скрывать неуспешное выполнение операции.

Особое внимание необходимо уделить контексту исполнения.

По умолчанию процедура может выполняться в контексте вызывающего пользователя, однако SQL Server позволяет определить другой контекст через:

```sql
WITH EXECUTE AS
```

Например:

```sql
WITH EXECUTE AS OWNER
```

Это позволяет создать контролируемый интерфейс, когда пользователю разрешено выполнить определенную процедуру, но прямой доступ к исходным таблицам не предоставляется.

При этом `EXECUTE AS OWNER` не следует добавлять ко всем процедурам автоматически. Изменение контекста исполнения влияет на модель безопасности базы данных и должно использоваться осознанно.

Для диагностики текущего контекста полезны:

```sql
ORIGINAL_LOGIN()
SUSER_SNAME()
USER_NAME()
```

После ручного:

```sql
EXECUTE AS USER = 'ProcedureTestUser';
```

обязательно возвращать исходный контекст:

```sql
REVERT;
```

Процедура не должна принимать параметры только потому, что это возможно. Каждый параметр должен иметь конкретное назначение.

Также необходимо правильно выбирать типы параметров. Если:

```text
CustomerId
```

в таблице имеет тип:

```sql
INT
```

то и параметр процедуры желательно объявлять:

```sql
@CustomerId INT
```

Если столбец:

```text
Email
```

имеет тип:

```sql
NVARCHAR(200)
```

не следует без причины создавать параметр другого несовместимого типа.

Имена процедур должны отражать выполняемую операцию.

Хороший вариант:

```text
usp_CreateRequest
usp_AssignEmployee
usp_ChangeRequestStatus
usp_GetRequestStatistics
```

Неудачные варианты:

```text
proc1
doRequest
testProc
procedure_new
```

Желательно придерживаться единой системы именования.

# Что проверить перед отправкой — чек-лист

## Структура базы

☐ Создана отдельная база данных.

☐ Создана схема `Support`.

☐ Созданы таблицы клиентов.

☐ Созданы таблицы сотрудников.

☐ Созданы категории.

☐ Созданы заявки.

☐ Создана история изменения статусов.

☐ Между таблицами настроены внешние ключи.

☐ Добавлены тестовые данные.

## Базовые процедуры

☐ Создана процедура без параметров.

☐ Создана процедура с одним параметром.

☐ Создана процедура с несколькими параметрами.

☐ Создана процедура с параметрами по умолчанию.

☐ Используется `SET NOCOUNT ON`.

## Добавление данных

☐ Создана процедура создания клиента.

☐ Создана процедура создания заявки.

☐ Используется `SCOPE_IDENTITY()`.

☐ Идентификатор возвращается через `OUTPUT`.

☐ Перед добавлением выполняется проверка входных данных.

## Изменение данных

☐ Создана процедура изменения клиента.

☐ Создана процедура назначения сотрудника.

☐ Создана процедура изменения статуса.

☐ Используется `@@ROWCOUNT`.

☐ Проверяются бизнес-условия.

## Удаление

☐ Реализована процедура удаления или деактивации.

☐ Учтены внешние ключи.

☐ Проверено физическое или логическое удаление.

## OUTPUT и RETURN

☐ Реализован один `OUTPUT`-параметр.

☐ Реализовано несколько `OUTPUT`-параметров.

☐ Использован `RETURN`.

☐ Проверен код возврата.

☐ Проверена разница между `SELECT`, `OUTPUT` и `RETURN`.

## Транзакции

☐ Есть процедура, изменяющая несколько таблиц.

☐ Используется `BEGIN TRANSACTION`.

☐ Используется `COMMIT`.

☐ Используется `ROLLBACK`.

☐ Используется `TRY...CATCH`.

☐ Используется `THROW`.

☐ Проверен `XACT_STATE()`.

☐ Исследован `SET XACT_ABORT ON`.

## Управление процедурами

☐ Использован `CREATE PROCEDURE`.

☐ Использован `ALTER PROCEDURE`.

☐ Использован `CREATE OR ALTER PROCEDURE`.

☐ Использован `DROP PROCEDURE`.

☐ Получен список через `sys.procedures`.

☐ Проверен `OBJECT_DEFINITION`.

☐ Проверены параметры через `sys.parameters`.

## Контекст исполнения

☐ Определен текущий пользователь.

☐ Проверен `ORIGINAL_LOGIN()`.

☐ Проверен `SUSER_SNAME()`.

☐ Проверен `USER_NAME()`.

☐ Выполнен эксперимент с `EXECUTE AS`.

☐ Использован `REVERT`.

☐ Создана процедура с `EXECUTE AS OWNER` или другим выбранным контекстом.

☐ Проверено выполнение процедуры без прямого доступа к таблице.

☐ Исследовано право `GRANT EXECUTE`.

## Производительность

☐ Одна процедура выполнена с разными параметрами.

☐ Использован `SET STATISTICS IO ON`.

☐ Использован `SET STATISTICS TIME ON`.

☐ Проверены планы выполнения.

☐ Исследована `sys.dm_exec_procedure_stats`.

☐ Сравнено несколько вариантов выполнения.

## Самостоятельная часть

☐ Разработана собственная комплексная процедура.

☐ Есть минимум два входных параметра.

☐ Есть проверка данных.

☐ Есть условная логика.

☐ Процедура работает минимум с двумя таблицами.

☐ Реализована обработка ошибок.

☐ При необходимости используется транзакция.

☐ Сформирована итоговая таблица процедур.

# Советы по улучшению работы

Не превращайте каждую SQL-команду в отдельную хранимую процедуру. Процедура должна иметь законченное назначение и представлять понятную операцию системы.

Хорошая процедура:

```text
создать заявку;
закрыть заявку;
назначить сотрудника;
получить отчет за период.
```

Менее удачный подход — создавать процедуру только ради выполнения одного технического действия без понятной прикладной роли.

Не доверяйте входным параметрам автоматически. Если процедура получает:

```sql
@CustomerId
```

необходимо определить, достаточно ли существования такого идентификатора или клиент дополнительно должен быть активен.

Если процедура получает:

```sql
@NewStatus
```

необходимо проверить, разрешен ли такой статус и допустим ли переход из текущего состояния.

Не дублируйте ограничения таблиц бессмысленно, но добавляйте в процедуру бизнес-проверки, которые невозможно выразить простым `CHECK` или внешним ключом.

Используйте именованные параметры при вызове процедур:

```sql
EXEC Support.usp_GetRequestReport
    @DateFrom = '2026-01-01',
    @DateTo = '2026-12-31',
    @Status = 'Closed';
```

Такой вызов значительно понятнее позиционного:

```sql
EXEC Support.usp_GetRequestReport
    '2026-01-01',
    '2026-12-31',
    'Closed',
    NULL;
```

Особенно это важно для процедур с большим количеством параметров.

Не используйте транзакцию вокруг огромного объема логики без необходимости. Чем дольше открыта транзакция, тем дольше могут удерживаться блокировки и тем выше вероятность конфликтов при параллельной работе.

Транзакция должна охватывать именно те изменения, которые обязаны быть атомарными.

Не скрывайте ошибки пустым `CATCH`.

Плохой вариант:

```sql
BEGIN CATCH
END CATCH;
```

или выполнение отката без передачи информации вызывающей стороне.

После необходимой обработки используйте:

```sql
THROW;
```

чтобы вызывающий код получил информацию об ошибке.

При разработке процедур изменения данных продумывайте возможность повторного вызова. Если операция повторяется, необходимо понимать, создаст ли она дубликат, завершится ли ошибкой или корректно определит, что требуемое состояние уже достигнуто.

Не выдавайте приложению или пользователю права на все таблицы только потому, что это проще. Хранимые процедуры могут выступать контролируемым интерфейсом изменения и чтения данных. Пользователь может получать право:

```sql
GRANT EXECUTE
```

на конкретные операции вместо широкого прямого доступа к таблицам.

При использовании:

```sql
WITH EXECUTE AS OWNER
```

обязательно анализируйте последствия для безопасности. Процедура может получить больше возможностей, чем вызывающий пользователь, поэтому ее код должен быть особенно тщательно ограничен.

При оценке производительности параметризованных процедур проверяйте выполнение с различными значениями параметров. Запрос по клиенту с одной заявкой и запрос по клиенту с десятками тысяч заявок могут предъявлять разные требования к плану выполнения.

Не создавайте одну огромную процедуру, которая в зависимости от десятков параметров выполняет совершенно разные операции. Процедуры должны оставаться понятными, тестируемыми и предсказуемыми.

Итоговая работа считается качественно выполненной, если созданные процедуры не только выполняются без ошибок, но и формируют понятный программный интерфейс базы данных: параметры имеют обоснованные типы, входные значения проверяются, связанные изменения защищены транзакциями, ошибки корректно передаются вызывающей стороне, а права и контекст исполнения соответствуют назначению операции.
