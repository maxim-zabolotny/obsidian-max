[[Кермит]]
[[Postgres]]

Добавлена: 2024-07-04

## Описание
Получил из меги

1 таблица:
- data -- 21 474 856

**Поиск по:** nickName, email
**Merge by:**  email


## Нормализация

## Индексы

```
  
-- Enable trgram  
CREATE EXTENSION IF NOT EXISTS pg_trgm;  
  
create index data__username__gin_idx  
   on data using gin (username gin_trgm_ops);

create index data__email  
    on data (email);

```

## Модификации


### **Удаление пустых строк **
```
delete from data where hash is null and salt is null and email is null and ip is null;

```

### **Удаление дублей**
  
```
-- Создаем временную таблицу для хранения строк, которые будут удалены  
CREATE TEMP TABLE to_be_deleted AS  
WITH duplicates AS (  
    SELECT  
        id,  
        email,  
        username,  
        hash,  
        tag,  
        ROW_NUMBER() OVER (PARTITION BY email, username, hash, tag ORDER BY id) AS rnum  
    FROM data  
)  
SELECT id, email, username, hash, tag  
FROM duplicates  
WHERE rnum > 1;  
  
DELETE FROM data  
WHERE id IN (SELECT id FROM to_be_deleted);  
  
-- Проверка содержимого временной таблицы  
SELECT * FROM to_be_deleted;
```


**Добавление столбца password** - Пока пустое но потом его заполнят дешифроваными паролями 

```
alter table data  
    add password varchar;
```

## Получение деталки

```
```



