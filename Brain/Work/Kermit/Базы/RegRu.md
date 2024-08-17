[[Кермит]]
[[Postgres]]

Добавлена: 2024-06-18

## Описание
3 таблицы:
- domains -- 21 474 856
- user_info -- 29 153 565
- users -- 7 263 166
- user_info_transformed - Новая таблица с нормализированными данными из user_info. В ней для 1 юзера 1 строка

**Поиск по:** domainName, email, phone, fio + dob
**Merge by:**  fio, email, phone


## Нормализация

Нужно делать нормализацию чтобы не тормозил мерж после получения деталки
	Идеальный вариант сделать одну таблицу, но есть сложность с табл user_info - в ней 54. варианта полей, причем с пересекающими типами например почта но имена по разному


## Индексы

```
create index user_info__user_id on user_info(user_id);  

create index domains__dname on domains(dname);  
create index domains__user_id on domains(user_id);  

  
create index users__phone on users(phone);  
create index users__email on users(email);  
create index users__hosting_email on users(hosting_email);  
create index users__user_id on users(user_id);

----------------------------------------
-- user_info_transformed
CREATE OR REPLACE FUNCTION replace_yo(text)  
RETURNS text  
IMMUTABLE  
LANGUAGE sql  
AS $$  
    SELECT translate($1, 'Ёё', 'Ее')  
$$;  
  
CREATE INDEX  user_info_transformed__replace_yo_fullname  
ON user_info_transformed (  
   replace_yo(person_r_name::text),  
   replace_yo(person_r_patronimic::text),  
   replace_yo(person_r_surname::text)  
);

  
-- Enable trgram  
CREATE EXTENSION IF NOT EXISTS pg_trgm;  
  
create index domains__dname__gin_idx  
   on domains using gin (dname gin_trgm_ops);

```

## Модификации

### **Нормализация номера в users**
```
UPDATE users  
SET phone = regexp_replace(phone, '\D', '', 'g') where phone is not null;

```

### **Удаление дубликатов из users**
```
 WITH duplicates AS (  
    SELECT  
        u.ctid,  
        u.user_id,  
        ROW_NUMBER() OVER (PARTITION BY u.user_id ORDER BY u.user_id) as row_num  
    FROM users u  
    JOIN (  
        SELECT  
            user_id  
        FROM users  
        GROUP BY user_id  
        HAVING COUNT(*) > 1  
    ) du ON u.user_id = du.user_id  
),  
to_delete AS (  
    SELECT *  
    FROM duplicates  
    WHERE row_num > 1  
)  
DELETE FROM users  
USING to_delete  
WHERE users.ctid = to_delete.ctid;
```

### **Формирование новой таблицы user_info где для каждого юзера будет одна строка**
```

CREATE TABLE user_info_transformed (  
    user_id INTEGER PRIMARY KEY,  
    actt_e_mail VARCHAR(255),  
    address VARCHAR(255),  
    address_r VARCHAR(255),  
    address_r_addr VARCHAR(255),  
    address_r_area VARCHAR(255),  
    address_r_city VARCHAR(255),  
    address_r_zip VARCHAR(255),  
    birth_date VARCHAR(255),  
    code VARCHAR(255),  
    country VARCHAR(255),  
    descr VARCHAR(255),  
    e_mail VARCHAR(255),  
    email VARCHAR(255),  
    fax VARCHAR(255),  
    fax_no VARCHAR(255),  
    first_name VARCHAR(255),  
    first_name_en VARCHAR(255),  
    kpp VARCHAR(255),  
    last_name VARCHAR(255),  
    last_name_en VARCHAR(255),  
    o_first_name VARCHAR(255),  
    o_last_name VARCHAR(255),  
    org VARCHAR(255),  
    org_r VARCHAR(255),  
    p_addr VARCHAR(255),  
    p_addr_addr VARCHAR(255),  
    p_addr_area VARCHAR(255),  
    p_addr_city VARCHAR(255),  
    p_addr_country VARCHAR(255),  
    p_addr_r VARCHAR(255),  
    p_addr_r_addr VARCHAR(255),  
    p_addr_r_area VARCHAR(255),  
    p_addr_r_city VARCHAR(255),  
    p_addr_recipient VARCHAR(255),  
    p_addr_r_zip VARCHAR(255),  
    p_addr_zip VARCHAR(255),  
    passport VARCHAR(255),  
    passport_date VARCHAR(255),  
    passport_number VARCHAR(255),  
    passport_place VARCHAR(255),  
    patronimic VARCHAR(255),  
    patronimic_en VARCHAR(255),  
    person VARCHAR(255),  
    person_name VARCHAR(255),  
    person_patronimic VARCHAR(255),  
    person_r_name VARCHAR(255),  
    person_r_patronimic VARCHAR(255),  
    person_r_surname VARCHAR(255),  
    person_surname VARCHAR(255),  
    phone VARCHAR(255),  
    private_person_flag VARCHAR(255),  
    sms_security_number VARCHAR(255),  
    total_pp_flag VARCHAR(255),  
    transfer_email VARCHAR(255)  
);  
  
--------------  
INSERT INTO user_info_transformed (user_id, actt_e_mail, address, address_r, address_r_addr, address_r_area, address_r_city, address_r_zip, birth_date, code, country, descr, e_mail, email, fax, fax_no, first_name, first_name_en, kpp, last_name, last_name_en, o_first_name, o_last_name, org, org_r, p_addr, p_addr_addr, p_addr_area, p_addr_city, p_addr_country, p_addr_r, p_addr_r_addr, p_addr_r_area, p_addr_r_city, p_addr_recipient, p_addr_r_zip, p_addr_zip, passport, passport_date, passport_number, passport_place, patronimic, patronimic_en, person, person_name, person_patronimic, person_r_name, person_r_patronimic, person_r_surname, person_surname, phone, private_person_flag, sms_security_number, total_pp_flag, transfer_email)  
SELECT  
    user_id,  
    MAX(CASE WHEN fldid = 'actt_e_mail' THEN value END) AS actt_e_mail,  
    MAX(CASE WHEN fldid = 'address' THEN value END) AS address,  
    MAX(CASE WHEN fldid = 'address_r' THEN value END) AS address_r,  
    MAX(CASE WHEN fldid = 'address_r_addr' THEN value END) AS address_r_addr,  
    MAX(CASE WHEN fldid = 'address_r_area' THEN value END) AS address_r_area,  
    MAX(CASE WHEN fldid = 'address_r_city' THEN value END) AS address_r_city,  
    MAX(CASE WHEN fldid = 'address_r_zip' THEN value END) AS address_r_zip,  
    MAX(CASE WHEN fldid = 'birth_date' THEN value END) AS birth_date,  
    MAX(CASE WHEN fldid = 'code' THEN value END) AS code,  
    MAX(CASE WHEN fldid = 'country' THEN value END) AS country,  
    MAX(CASE WHEN fldid = 'descr' THEN value END) AS descr,  
    MAX(CASE WHEN fldid = 'e_mail' THEN value END) AS e_mail,  
    MAX(CASE WHEN fldid = 'email' THEN value END) AS email,  
    MAX(CASE WHEN fldid = 'fax' THEN value END) AS fax,  
    MAX(CASE WHEN fldid = 'fax_no' THEN value END) AS fax_no,  
    MAX(CASE WHEN fldid = 'first_name' THEN value END) AS first_name,  
    MAX(CASE WHEN fldid = 'first_name_en' THEN value END) AS first_name_en,  
    MAX(CASE WHEN fldid = 'kpp' THEN value END) AS kpp,  
    MAX(CASE WHEN fldid = 'last_name' THEN value END) AS last_name,  
    MAX(CASE WHEN fldid = 'last_name_en' THEN value END) AS last_name_en,  
    MAX(CASE WHEN fldid = 'o_first_name' THEN value END) AS o_first_name,  
    MAX(CASE WHEN fldid = 'o_last_name' THEN value END) AS o_last_name,  
    MAX(CASE WHEN fldid = 'org' THEN value END) AS org,  
    MAX(CASE WHEN fldid = 'org_r' THEN value END) AS org_r,  
    MAX(CASE WHEN fldid = 'p_addr' THEN value END) AS p_addr,  
    MAX(CASE WHEN fldid = 'p_addr_addr' THEN value END) AS p_addr_addr,  
    MAX(CASE WHEN fldid = 'p_addr_area' THEN value END) AS p_addr_area,  
    MAX(CASE WHEN fldid = 'p_addr_city' THEN value END) AS p_addr_city,  
    MAX(CASE WHEN fldid = 'p_addr_country' THEN value END) AS p_addr_country,  
    MAX(CASE WHEN fldid = 'p_addr_r' THEN value END) AS p_addr_r,  
    MAX(CASE WHEN fldid = 'p_addr_r_addr' THEN value END) AS p_addr_r_addr,  
    MAX(CASE WHEN fldid = 'p_addr_r_area' THEN value END) AS p_addr_r_area,  
    MAX(CASE WHEN fldid = 'p_addr_r_city' THEN value END) AS p_addr_r_city,  
    MAX(CASE WHEN fldid = 'p_addr_recipient' THEN value END) AS p_addr_recipient,  
    MAX(CASE WHEN fldid = 'p_addr_r_zip' THEN value END) AS p_addr_r_zip,  
    MAX(CASE WHEN fldid = 'p_addr_zip' THEN value END) AS p_addr_zip,  
    MAX(CASE WHEN fldid = 'passport' THEN value END) AS passport,  
    MAX(CASE WHEN fldid = 'passport_date' THEN value END) AS passport_date,  
    MAX(CASE WHEN fldid = 'passport_number' THEN value END) AS passport_number,  
    MAX(CASE WHEN fldid = 'passport_place' THEN value END) AS passport_place,  
    MAX(CASE WHEN fldid = 'patronimic' THEN value END) AS patronimic,  
    MAX(CASE WHEN fldid = 'patronimic_en' THEN value END) AS patronimic_en,  
    MAX(CASE WHEN fldid = 'person' THEN value END) AS person,  
    MAX(CASE WHEN fldid = 'person_name' THEN value END) AS person_name,  
    MAX(CASE WHEN fldid = 'person_patronimic' THEN value END) AS person_patronimic,  
    MAX(CASE WHEN fldid = 'person_r_name' THEN value END) AS person_r_name,  
    MAX(CASE WHEN fldid = 'person_r_patronimic' THEN value END) AS person_r_patronimic,  
    MAX(CASE WHEN fldid = 'person_r_surname' THEN value END) AS person_r_surname,  
    MAX(CASE WHEN fldid = 'person_surname' THEN value END) AS person_surname,  
    MAX(CASE WHEN fldid = 'phone' THEN value END) AS phone,  
    MAX(CASE WHEN fldid = 'private_person_flag' THEN value END) AS private_person_flag,  
    MAX(CASE WHEN fldid = 'sms_security_number' THEN value END) AS sms_security_number,  
    MAX(CASE WHEN fldid = 'total_pp_flag' THEN value END) AS total_pp_flag,  
    MAX(CASE WHEN fldid = 'transfer_email' THEN value END) AS transfer_email  
FROM user_info  
GROUP BY user_id;
```

### **Нормализация Даты рождения user_info_transformed**
```
ALTER TABLE user_info_transformed ADD COLUMN birth_date_new DATE;  
UPDATE user_info_transformed  
SET birth_date_new = TO_DATE(birth_date, 'DD.MM.YYYY');  
  
ALTER TABLE user_info_transformed DROP COLUMN birth_date;  
ALTER TABLE user_info_transformed RENAME COLUMN birth_date_new TO birth_date;

```


### **Нормализация номера в users**
```
UPDATE user_info_transformed  
SET phone = regexp_replace(phone, '\D', '', 'g') where phone is not null;
```

### Телефон в число
```
ALTER TABLE users ADD COLUMN phone_numeric NUMERIC;  
update users set phone = null where phone = '';  
UPDATE users  
SET phone_numeric = REGEXP_REPLACE(phone, '\D', '', 'g')::NUMERIC where phone is not null;  
ALTER TABLE users DROP COLUMN phone;  
ALTER TABLE users RENAME COLUMN phone_numeric TO phone;
```

## Получение деталки

```
SELECT 
  u.user_id as id, 
  u."lastlogin", 
  u.email, 
  u.hosting_email, 
  u.phone, 
  CONCAT(
    ui.p_addr_city, ui.address_r_addr
  ) as address, 
  p_addr_recipient as recipient, 
  passport_date as passport_issue, 
  passport_number, 
  passport_place, 
  person_r_name as firstname, 
  person_r_patronimic as patronymic, 
  person_r_surname as lastname, 
  birth_date as dob, 
  ui.phone as phone_ui, 
  array_agg(DISTINCT d.dname) as domains 
FROM 
  users u 
  left join user_info_transformed ui on ui.user_id = u.user_id 
  left join public.domains d on u.user_id = d.user_id 
WHERE 
  u.user_id = ANY($1 :: bigint[]) 
GROUP BY 
  u.user_id, 
  u."lastlogin", 
  u.email, 
  u.hosting_email, 
  u.phone, 
  ui.p_addr_city, 
  ui.address_r_addr, 
  ui.p_addr_recipient, 
  ui.passport_date, 
  ui.passport_number, 
  ui.passport_place, 
  ui.person_r_name, 
  ui.person_r_patronimic, 
  ui.person_r_surname, 
  ui.birth_date, 
  ui.phone

```



