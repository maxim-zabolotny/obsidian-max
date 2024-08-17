[[Кермит]]
[[Postgres]]
## Информация: 

**Уникальных вин номеров:**   
**Уникальных номеров:** 

```
select count(*) from vehicle; -- 579 837 275   
```

```
select count(*) from vehicle where vin is null; -- 58 021 772
```

```
select count(*) from vehicle where vin is null and license_plate is null; -- 9 692 562
```

```
select count(*) from vehicle where vin is null and license_plate is not null; -- 48 329 210
```

```
SELECT COUNT(DISTINCT vin) AS unique_vin_count FROM vehicle; -- 87 017 919
```
