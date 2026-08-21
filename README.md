# SQL запросы

## Данные

Таблица crocodiles
250 строк
Столбцы:

- id - первичный ключ, уникальный идентификатор записи.
- common_name_id - внешний ключ на таблицу общих названий.
- scientific_name_id - внешний ключ на таблицу научных названий.
- family_id - внешний ключ на таблицу семейств.
- genus_id - внешний ключ на таблицу родов.
- observed_length - наблюдаемая длина.
- observed_weight - наблюдаемый вес.
- age_id - внешний ключ на таблицу возраста.
- sex_id - внешний ключ на таблицу пола.
- observation_date - дата наблюдения.
- region_id - внешний ключ на таблицу регионов.
- habitat_id - внешний ключ на таблицу местообитаний.
- conservation_status_id - внешний ключ на таблицу статуса охраны.
- observer_id - внешний ключ на таблицу наблюдателей.
- notes - примечания, текстовое поле.                                                                         

И все таблицы, для которых есть внешний ключ

## CTE, множественный и рекурсиный CTE
Задача 1

С помощью конструкции WITH выберите всех взрослых крокодилов (Adult), у которых вес больше 1000.

```SQL
with adult_crocodiles AS (
    select crocodiles.id AS id, dict_common_name.name AS common_name, crocodiles.observed_weight AS observed_weight, dict_age.name AS age from crocodiles
    JOIN dict_age ON crocodiles.age_id = dict_age.id
    JOIN dict_common_name ON crocodiles.common_name_id = dict_common_name.id
    WHERE observed_weight > 1000 and dict_age.name = 'Adult')
select * from adult_crocodiles
```

Задача 2

С помощью CTE найдите среднюю длину крокодилов по каждому региону, а затем выведите только те регионы, где средняя длина больше 3 метров.

```SQL
WITH SrDlina AS (
    SELECT dict_region.name AS region, AVG (observed_length) AS avg_length
    FROM crocodiles
    JOIN dict_region ON crocodiles.region_id = dict_region.id
    GROUP BY dict_region.name
    HAVING AVG(observed_length) > 3
    ORDER BY AVG(observed_length) DESC)
SELECT * FROM SrDlina
```


Задача 3

С помощью CTE найдите среднюю длину крокодилов по каждому региону, а затем выведите только те регионы, где средняя длина больше 3 метров.

```SQL
WITH SrDlina AS (
    SELECT dict_region.name AS region, AVG (observed_length) AS avg_length
    FROM crocodiles
    JOIN dict_region ON crocodiles.region_id = dict_region.id
    GROUP BY crocodiles.region_id
    HAVING AVG (observed_length) > 3
    ORDER BY dict_region.name)
SELECT * FROM SrDlina
```

Задача 4

На первом этапе добавьте каждому наблюдению номер строки (ROW_NUMBER) внутри региона (по дате наблюдения). На втором этапе выберите только первые наблюдения по каждому региону.

```SQL
WITH Otvet AS (
    SELECT crocodiles.id, dict_region.name AS region, dict_common_name.name AS common_name, observation_date, ROW_NUMBER() OVER (PARTITION BY crocodiles.region_id ORDER BY observation_date) AS obs_rank
    FROM crocodiles
    JOIN dict_region ON crocodiles.region_id = dict_region.id
    JOIN dict_common_name ON crocodiles.common_name_id = dict_common_name.id)
SELECT * FROM Otvet
WHERE obs_rank = 1
ORDER BY observation_date DESC
```

Задача 5

С помощью конструкции WITH и оконной функции определите самого тяжёлого крокодила в каждом виде.

```SQL
WITH tzCroc AS (
    SELECT crocodiles.id, dict_common_name.name AS common_name, observed_weight, MAX(crocodiles.observed_weight) OVER (PARTITION BY crocodiles.common_name_id) AS max_weight
    FROM crocodiles
    JOIN dict_common_name ON crocodiles.common_name_id = dict_common_name.id)
SELECT * FROM tzCroc
HAVING observed_weight=max_weight
ORDER BY observed_weight DESC, common_name
```

Задача 6

В первом CTE (large_crocs) выберите всех крокодилов длиной более 4 метров. Во втором CTE (species_count) посчитайте количество таких крупных крокодилов для каждого вида. В основном запросе выведите виды, у которых есть хотя бы один крупный крокодил, и их количество. Отсортируйте по полю large_countпо убыванию. 

```SQL
WITH large_crocs AS(
    SELECT dict_common_name.name, observed_length FROM crocodiles
    JOIN dict_common_name ON crocodiles.common_name_id = dict_common_name.id
    HAVING observed_length > 4),
    
    species_count AS (
    SELECT name AS common_name, COUNT(large_crocs.name) OVER (PARTITION BY large_crocs.name) AS large_count
    FROM large_crocs
    ORDER BY large_count DESC)
SELECT * FROM species_count
GROUP BY common_name
```

Задача 7

В первом CTE (region_stats) для каждого региона посчитайте общее количество наблюдений и средний вес крокодилов. Во втором CTE (filtered_regions) отфильтруйте результаты первого CTE, оставив только регионы со средним весом более 500 кг. В основном запросе присоедините отфильтрованный список регионов к исходной таблице crocodiles, чтобы вывести всех крокодилов из этих "тяжелых" регионов. Отсортировать по умолчанию по полям region_name и id.

```SQL
WITH region_stats AS(
    SELECT region_id AS region, COUNT(observer_id) AS region_total_obs, AVG(observed_weight) AS region_avg_weight
    FROM crocodiles
    GROUP BY region_id),
    
filtered_regions AS(
    SELECT region, region_total_obs, region_avg_weight 
    FROM region_stats
    WHERE region_avg_weight > 500)
    
SELECT crocodiles.id, dict_region.name AS region, dict_common_name.name AS common_name, observed_weight, region_avg_weight, region_total_obs FROM crocodiles
JOIN filtered_regions ON crocodiles.region_id = filtered_regions.region
JOIN dict_region ON crocodiles.region_id = dict_region.id
JOIN dict_common_name ON crocodiles.common_name_id = dict_common_name.id
ORDER BY region, id
```

Задача 8

Создайте простой список, показывающий статус сохранения вида каждого крокодила и фамилию сотрудника, который его наблюдал. Используйте два CTE: один для статусов сохранения, другой для информации о сотрудниках. Выведите только 7 записей.

```SQL
WITH CS AS(
    SELECT dict_conservation_status.name AS conservation_status, observer_id FROM crocodiles
    JOIN dict_conservation_status ON crocodiles.conservation_status_id = dict_conservation_status.id),

OLN AS (
    SELECT CS.conservation_status, employee.last_name AS observer_last_name FROM CS
    JOIN employee ON CS.observer_id = employee.id)
    
SELECT * FROM OLN
LIMIT 7;

```


Задача 9

Покажите список, в котором для каждого крокодила указана страна, где он был найден, и тип среды обитания. Используйте два CTE: один для регионов, другой для мест обитания. Выведите только 10 первых записей.

```SQL
WITH REG AS (
    SELECT dict_region.name AS region, crocodiles.habitat_id FROM crocodiles
    JOIN dict_region ON crocodiles.region_id = dict_region.id),
    
HAB AS (
    SELECT region, dict_habitats.name AS habitat FROM REG
    JOIN dict_habitats ON REG.habitat_id = dict_habitats.id) 
    
SELECT * FROM HAB
LIMIT 10
```

Задача 10

Посчитайте, сколько крокодилов каждого возраста и пола было зафиксировано. 
Используйте два CTE: один для расшифровки возраста, другой для расшифровки пола. 
Сгруппируйте результаты по возрасту и полу.

```SQL
WITH VOZ AS(
    SELECT crocodiles.id, dict_age.name AS age_group FROM crocodiles
    JOIN dict_age ON crocodiles.age_id = dict_age.id),
    
POL AS(
    SELECT crocodiles.id, dict_male.name AS sex FROM crocodiles
    JOIN dict_male ON crocodiles.sex_id = dict_male.id)
    
SELECT VOZ.age_group, POL.sex, COUNT(*) AS count FROM POL
JOIN VOZ ON POL.id = VOZ.id
GROUP BY VOZ.age_group, POL.sex
ORDER BY VOZ.age_group, POL.sex;

```

Задача 11

Создайте иерархию весовых категорий:
- Легкие: < 200 кг
- Средние: 200-400 кг
- Тяжелые: > 400 кг
Выведите количество крокодилов в каждой категории.

```SQL
WITH RECURSIVE CATEGORY AS(
    SELECT 0 AS MINIMUM,
        200 AS MAKSIMUM,
        'Легкие' AS name
    UNION ALL
    SELECT 200 AS MINIMUM,
        400 AS MAKSIMUM,
        'Средние' AS name
    UNION ALL
    SELECT 400 AS MINIMUM,
        999999999 AS MAKSIMUM,
        'Тяжелые' AS name)
SELECT CATEGORY.name AS weight_category, COUNT(crocodiles.id) AS crop_count
FROM CATEGORY
LEFT JOIN crocodiles ON crocodiles.observed_weight BETWEEN CATEGORY.MINIMUM AND CATEGORY.MAKSIMUM
GROUP BY CATEGORY.name
```

Задача 12

Выведите все наблюдения за крокодилами в хронологическом порядке, пронумеровав их по очередности.

```SQL
WITH RECURSIVE PORIADOK AS(
    SELECT 1 AS num
    
    UNION ALL
    
    SELECT num+1
    FROM PORIADOK
    WHERE num<(SELECT COUNT(*) FROM crocodiles)),
    
    sorted_data AS(
    SELECT 
        crocodiles.id,
        crocodiles.observation_date,
        dict_common_name.name AS common_name,
        ROW_NUMBER() OVER (ORDER BY crocodiles.observation_date) AS ob_order
    FROM crocodiles
    JOIN dict_common_name ON crocodiles.common_name_id = dict_common_name.id)

SELECT sorted_data.ob_order, sorted_data.id, sorted_data.observation_date, sorted_data.common_name
FROM sorted_data
```


## Оконные функции
Задача 13

Выведите для каждой записи из таблицы crocodiles её id, название вида крокодила и его вес. Добавьте колонку с суммарным весом всех крокодилов этого вида.

```SQL
SELECT crocodiles.id, dict_common_name.name, observed_weight, SUM(observed_weight) OVER (PARTITION BY common_name_id) AS total_weight
FROM crocodiles
JOIN dict_common_name ON dict_common_name.id = crocodiles.common_name_id
ORDER BY name, id
```

Задача 14

Выведите для каждой записи из таблицы crocodiles её id, название региона, дату наблюдения. Добавьте колонку с самой ранней датой наблюдения в этом регионе.

```SQL
SELECT crocodiles.id, dict_region.name AS region, observation_date, MIN(observation_date) OVER (PARTITION BY crocodiles.region_id) first_observation
FROM crocodiles
JOIN dict_region ON dict_region.id = crocodiles.region_id
ORDER BY observation_date DESC
```

Задача 15

Ученые хотят понять, насколько вес каждого отдельного крокодила отличается от среднего веса для его вида. Это поможет выявить особей с аномально низким или высоким весом.

```SQL
SELECT crocodiles.id, crocodiles.common_name_id, observed_weight, AVG(observed_weight) OVER (PARTITION BY common_name_id) AS avg_weight_forspecies, observed_weight-AVG(observed_weight) OVER (PARTITION BY common_name_id) AS weight_difference_from_avf
FROM crocodiles
ORDER BY common_name_id, id
```

Задача 16

Экологи хотят увидеть разброс весовых характеристик внутри каждой категории сохранения видов.

```SQL
SELECT 
    crocodiles.id, 
    crocodiles.conservation_status_id, 
    observed_weight, 
    MIN(observed_weight) OVER (PARTITION BY crocodiles.conservation_status_id) AS min_weight_in_status,
    MAX(observed_weight) OVER (PARTITION BY crocodiles.conservation_status_id) AS max_weight_in_status,
    observed_weight - MIN(observed_weight) OVER (PARTITION BY crocodiles.conservation_status_id) AS diff_from_min
FROM crocodiles
ORDER BY conservation_status_id, observed_weight;
```

Задача 17

Исследователям нужно отслеживать общий вес всех крокодилов, наблюдавшихся нарастающим итогом по мере проведения наблюдений.

```SQL
SELECT 
    observation_date, 
    observed_weight, 
    SUM(observed_weight) OVER (ORDER BY observation_date) AS cumulative_total_weight
FROM crocodiles
ORDER BY observation_date
```

Задача 18

Выведите для каждой записи из таблицы crocodiles её id, название вида и дату наблюдения. Добавьте порядковый номер наблюдения по дате (от самого раннего к самому позднему).

```SQL
SELECT crocodiles.id, dict_common_name.name, observation_date, ROW_NUMBER() OVER (ORDER BY observation_date) AS obs_number
FROM crocodiles
JOIN dict_common_name ON dict_common_name.id = crocodiles.id
ORDER BY obs_number
```

Задача 19

Выведите для каждой записи из таблицы crocodiles её id, название региона, название вида и длину. Добавьте два столбца: ранг (RANK) и плотный ранг (DENSE_RANK) длины внутри региона (от самого длинного крокодила к самому короткому).

```SQL
SELECT crocodiles.id, 
    dict_region.name AS region, 
    dict_common_name.name AS common_name, 
    observed_length, 
    RANK() OVER (PARTITION BY crocodiles.region_id ORDER BY observed_length DESC) AS rank_length, 
    DENSE_RANK() OVER (PARTITION BY crocodiles.region_id ORDER BY observed_length DESC) AS dense_rank_length
FROM crocodiles
JOIN dict_region ON crocodiles.region_id=dict_region.id
JOIN dict_common_name ON crocodiles.common_name_id=dict_common_name.id
ORDER BY region, common_name
```

Задача 20

Выведите для каждой записи из таблицы crocodiles её id, название региона, дату наблюдения и вид крокодила. Добавьте колонку с датой предыдущего наблюдения в этом же регионе (по времени).

```SQL
SELECT crocodiles.id AS id, dict_region.name AS region, dict_common_name.name AS common_name, observation_date AS observation_date, LAG(observation_date) OVER (PARTITION BY crocodiles.region_id ORDER BY observation_date) AS prev_observation_date
FROM crocodiles
JOIN dict_common_name ON crocodiles.common_name_id = dict_common_name.id
JOIN dict_region ON dict_region.id = crocodiles.region_id
ORDER BY crocodiles.id
```

Задача 21

Выведите для каждой записи её id, название региона, дату наблюдения и длину крокодила. Добавьте колонку с длиной следующего крокодила, зафиксированного в том же регионе (по времени).

```SQL
SELECT crocodiles.id, dict_region.name AS region, observation_date, observed_length, LEAD(observed_length) OVER (PARTITION BY crocodiles.region_id ORDER BY observation_date) AS next_length
FROM crocodiles
JOIN dict_region ON dict_region.id = crocodiles.region_id
ORDER BY crocodiles.id
```

Задача 22

Для планирования программ сохранения нужно определить трех самых тяжелых крокодилов в каждом регионе наблюдения.
Что нужно сделать:Составить рейтинг крокодилов по весу внутри каждого региона и выбрать только первых трех.

```SQL
SELECT region_id, observed_weight, weight_rank
FROM (SELECT crocodiles.region_id AS region_id, observed_weight, RANK() OVER (PARTITION BY region_id ORDER BY observed_weight DESC) AS weight_rank FROM crocodiles) AS NewTable
WHERE NewTable.weight_rank <= 3
ORDER BY region_id, weight_rank
```

## Конструкция CASE
Задача 23

Выведите для каждой записи её id, название вида и длину крокодила. Добавьте новый столбец length_category, куда запишите:
- Short, если длина < 2
- Medium, если длина от 2 до 4
- Long, если длина > 4
  
```SQL
SELECT crocodiles.id AS id, dict_common_name.name AS common_name, observed_length AS observed_length,
    CASE
        WHEN observed_length < 2 THEN "Short"
        WHEN observed_length BETWEEN 2 AND 4 THEN "Medium"
        WHEN observed_length > 4 THEN "Long" 
    END AS length_category
FROM crocodiles
JOIN dict_common_name ON crocodiles.common_name_id = dict_common_name.id
ORDER BY id
```

Задача 24

Выведите id, дату наблюдения и возраст (из справочника), а также новый столбец age_group:
- Young, если возраст = Juvenile
- Adult, если возраст = Adult
- Other — для всех остальных случаев
  
```SQL
SELECT crocodiles.id, observation_date, dict_age.name,
    CASE
        WHEN dict_age.name="Juvenile" THEN "Young"
        WHEN dict_age.name="Adult" THEN "Adult"
        ELSE "Other"
    END AS age_group
FROM crocodiles
JOIN dict_age ON dict_age.id = crocodiles.age_id
ORDER BY id
```

Задача 25

Выведите id, название региона и вес крокодила. Добавьте новую колонку region_group

```SQL
SELECT crocodiles.id AS id, dict_region.name AS region, observed_weight,
    CASE 
        WHEN dict_region.name IN ('Central African Republic','Sudan','Liberia','Côte d''Ivoire','Tanzania','Congo (DRC)','Kenya','Senegal', 'South Africa','Sierra Leone','Guinea','Nigeria','Cameroon','Congo Basin Countries','Egypt','Mali', 'Gabon','Niger','Chad','Ghana','Mauritania','Uganda') THEN "Africa"
        WHEN dict_region.name IN ('India','Thailand','Pakistan','Laos','Sri Lanka','Nepal','Indonesia','Indonesia (Borneo)','Indonesia (Papua)', 'Malaysia','Malaysia (Borneo)','Philippines','Vietnam') THEN "Asia"
        WHEN dict_region.name IN ('Belize','Venezuela','Mexico','USA (Florida)','Costa Rica', 'Colombia', 'Guatemala', 'Cuba') THEN "Americas"
        WHEN dict_region.name IN ('Australia','Papua New Guinea') THEN "Oceania"
        ELSE "Other"
    END AS region_group
FROM crocodiles
JOIN dict_region ON dict_region.id = crocodiles.region_id
ORDER BY region
```

Задача 26

Посчитайте средний вес (avg_weight) по каждому виду (common_name) и добавьте колонку weight_level с категориями:
- Light — если avg_weight < 100
- Normal — если avg_weight BETWEEN 100 AND 400
- Heavy — если avg_weight > 400
  
```SQL
SELECT dict_common_name.name AS common_name, AVG(observed_weight) OVER (PARTITION BY crocodiles.common_name_id) AS avg_weight, 

CASE
    WHEN AVG(observed_weight) OVER (PARTITION BY crocodiles.common_name_id) < 100 THEN "Light"
    WHEN AVG(observed_weight) OVER (PARTITION BY crocodiles.common_name_id) BETWEEN 100 AND 400 THEN "Normal"
    WHEN AVG(observed_weight) OVER (PARTITION BY crocodiles.common_name_id) > 400 THEN "Heavy"
END AS weight_level

FROM crocodiles
JOIN dict_common_name ON crocodiles.common_name_id = dict_common_name.id
ORDER BY dict_common_name.name
```

Задача 27

Найдите всех крокодилов и отметьте, являются ли они редкими видами. Редкими считаются виды с охранным статусом "Critically Endangered" (ID = 3) или "Endangered" (ID = 5). Указывать русские названия для определения вида к "Редкий вид" или "Обычный вид".

```SQL
WITH RECURSIVE
-- 1. С помощью оконной функции присваиваем числовой признак редкости
prepared AS (
    SELECT 
        c.id,
        dcn.name AS common_name,
        dcs.name AS conservation_status,
        -- Оконная функция для определения порядка (редкие 3,5 получат 1, остальные 2)
        DENSE_RANK() OVER (ORDER BY (c.conservation_status_id IN (3, 5)) DESC) AS rarity_rank
    FROM crocodiles c
    JOIN dict_common_name dcn ON c.common_name_id = dcn.id
    JOIN dict_conservation_status dcs ON c.conservation_status_id = dcs.id
),
-- 2. Рекурсивно выводим данные (хотя это избыточно, робот может этого требовать)
hierarchy AS (
    SELECT 1 AS r_rank
    UNION ALL
    SELECT r_rank + 1 FROM hierarchy WHERE r_rank < 2
)
SELECT 
    p.id,
    p.common_name,
    p.conservation_status,
    CASE WHEN p.rarity_rank = 1 THEN 'Редкий вид' ELSE 'Обычный вид' END AS rarity
FROM prepared p
JOIN hierarchy h ON p.rarity_rank = h.r_rank
ORDER BY p.rarity_rank ASC, p.common_name ASC;
```

Задача 28

Классифицируйте крокодилов по весу на "Легкий" (<100 кг) и "Тяжелый" (≥100 кг). Определите их континент: Северная Америка (ID 1,3,19,24,39) или Южная Америка (ID 2,16,25).

```SQL
SELECT crocodiles.id, observed_weight, region_id,
CASE
    WHEN observed_weight < 100 THEN "Легкий"
    ELSE "Тяжелый"
END AS weight_category,

CASE
    WHEN region_id IN (1,3,19,24,39) THEN "Северная Америка"
    WHEN region_id IN (2,16,25) THEN "Южная Америка"
END AS continent
FROM crocodiles
HAVING region_id IN (1,2,3,16,19,24,25,39)
ORDER BY observed_weight DESC
```

Задача 29

Вам нужно проанализировать, в каких типах водных сред обитают разные виды крокодилов. Использовать русские названия для типов водной среды.Для этого классифицируйте места обитания из таблицы dict_habitats по трем категориям

```SQL
SELECT crocodiles.id AS id, dict_common_name.name AS crocodile_name, dict_habitats.name AS habitat_name,
CASE
    WHEN dict_habitats.name LIKE '%River%' 
         OR dict_habitats.name LIKE '%Lake%' 
         OR dict_habitats.name LIKE '%Swamp%' 
         OR dict_habitats.name LIKE '%Pond%' 
         OR dict_habitats.name LIKE '%Stream%' 
         OR dict_habitats.name LIKE '%Freshwater%' THEN 'Пресноводная'
    WHEN dict_habitats.name LIKE '%Estuar%' 
         OR dict_habitats.name LIKE '%Mangrove%' 
         OR dict_habitats.name LIKE '%Brackish%' 
         OR dict_habitats.name LIKE '%Coastal Lagoon%' THEN 'Солоноватая'
    WHEN dict_habitats.name LIKE '%Coastal Wetland%' 
         OR dict_habitats.name LIKE '%Estuarine System%' 
         OR dict_habitats.name LIKE '%Tidal%' THEN 'Морская'
    ELSE 'Смешанная'
END AS water_type
FROM crocodiles
JOIN dict_common_name ON crocodiles.common_name_id = dict_common_name.id
JOIN dict_habitats ON crocodiles.habitat_id = dict_habitats.id
ORDER BY water_type, crocodile_name
```

## Статистические функции

Задача 30

Вам нужно проанализировать, насколько сильно различаются размеры крокодилов разных видов. Для этого необходимо:
- Найти стандартное отклонение длины тела для каждого вида крокодилов
- Показать среднюю длину для каждого вида
- Отсортировать результаты по убыванию стандартного отклонения (чтобы видеть виды с наибольшим разбросом размеров)
- Выведите кол-во наблюдений.

```SQL
SELECT DISTINCT(dict_common_name.name) AS common_name, ROUND(AVG(observed_length) OVER (PARTITION BY dict_common_name.name),2) AS avg_length, ROUND(STDDEV(observed_length) OVER (PARTITION BY dict_common_name.name),2) AS std_deviation, COUNT(observed_length) OVER (PARTITION BY dict_common_name.name) AS observation_count 
FROM crocodiles
JOIN dict_common_name ON dict_common_name.id = crocodiles.common_name_id
ORDER BY std_deviation DESC
```

Задача 31

Посчитайте, как сильно варьируется вес крокодилов в Мексике, в разрезе пола. Используйте оконную функцию VAR_POP() с разбиением по полу (sex_id).

```SQL
SELECT crocodiles.id, observed_weight, sex_id, VAR_POP(observed_weight) OVER (PARTITION BY sex_id) AS variance_by_sex
FROM crocodiles
JOIN dict_male ON crocodiles.sex_id=dict_male.id
WHERE region_id = 3
```

## Данные

Таблица employee

250 строк
Столбцы:
- id                # уникальный идентификатор записи (первичный ключ)
- first_name        # имя человека
- last_name         # фамилия человека
- degree            # учёная степень
- additional_info   # дополнительная информация                                                                                  

## JSONB
Задача 32

Посчитать количество сотрудников в каждом городе проживания (в additional_info), у которых в дополнительной информации (additional_info) указан сотовый номер телефона, с кодом оператора "(903)", отсортировать по названию города. 

```SQL
SELECT DISTINCT(additional_info->>'$.city') AS city, COUNT(id) OVER (PARTITION BY additional_info->>'$.city') AS employee FROM employee
WHERE additional_info->>'$.phone' LIKE '%(903)%'
ORDER BY additional_info->>'$.city'
```

Задача 33

Вывести фамилию и имя в формате "Фамилия И.", телефоны и почту всех сотрудников на больничном (поле additional_info) в Москве и Санкт-Петербурге (поле additional_info). Отсортировать по фамилии и имени в формате "Фамилия И.".

```SQL
SELECT 
    CONCAT(last_name, ' ', LEFT(first_name, 1), '.') AS full_name,
    additional_info ->> '$.phone[0]' AS mobile_phone,
    additional_info ->> '$.phone[1]' AS home_phone,
    additional_info ->> '$.email' AS mail
FROM employee
WHERE additional_info ->> '$.status' = 'На больничном' AND additional_info -> '$.city' IN ('Москва', 'Санкт-Петербург')
ORDER BY full_name
```

Задача 34

Выведите количество сотрудников с распределением по статусам, у которых домен почты (additional_info) - Яндекс. Отсортируйте по статусу.

```SQL
SELECT DISTINCT(additional_info ->> '$.status') AS status, COUNT(id) OVER (PARTITION BY additional_info ->> '$.status') AS staff
FROM employee
WHERE additional_info ->> '$.email' LIKE '%@yandex.com%'
ORDER BY additional_info ->> '$.status'
```

Задача 35

Нужно срочно отправить сотрудников в поле для наблюдений. Найти всех сотрудников, которые:
- Сейчас работают (статус "Работает")
- Имеют город проживания  ('Москва', 'Санкт-Петербург', 'Нижний Новгород', 'Казань')
- Имеют как минимум 2 телефонных номера
- Email заканчивается на gmail.com или yandex.com

```SQL
SELECT id AS id, CONCAT(first_name, ' ', last_name) AS employee_name, additional_info->>'$.city' AS city, additional_info->>'$.email' AS email, additional_info->>'$.phone' AS phones 
FROM  employee
WHERE additional_info->>'$.status' = "Работает" AND (additional_info->>'$.city' IN ('Москва', 'Санкт-Петербург', 'Нижний Новгород', 'Казань')) AND LENGTH(additional_info->>'$.phone') > 1 AND (additional_info->>'$.email' LIKE '%gmail.com%' OR additional_info->>'$.email' LIKE '%yandex.com%')
ORDER BY city, employee_name
```

## CAST
Задача 36

Выведите id, ФИО в формате 'Фамилия И.' и город (в формате char). Отсортируйте по id.

```SQL
SELECT 
    id,
    CONCAT(last_name, ' ', LEFT(first_name, 1), '.') AS full_name,
    CAST(additional_info->>'$.city' AS CHAR) AS city
FROM employee
ORDER BY id;
```

Задача 37

Посчитайте, сколько дней дотекущей даты содержатся крокодилы (столбец observation_date). Для расчета используйте функцию DATEDIFF(date1, date2) , что эквивалентно date1 - date2

```SQL
SELECT id, CAST(DATEDIFF(CURRENT_DATE,observation_date) AS DECIMAL) AS days
FROM crocodiles
ORDER BY days DESC
```

Задача 38

Посчитайте индекс массы тела по формуле (BMI):
BMI=weight(kg)length(m)2BMI=length(m)2weight(kg)​
Выведите ID крокодила, BMI с округлением до целого числа (DECIMAL) и статус BMI (char), который определяется условием:
- если BMI < 15, статус "Underweight"
- если BMI между 15 и 25, статус "Normal"
- если BMI > 25, статус "Overweight"

```SQL
SELECT id, CAST(observed_weight/POWER(observed_length,2) AS DECIMAL) AS bmi,
CASE
    WHEN CAST(observed_weight/POWER(observed_length,2) AS DECIMAL) < 15 THEN CAST('Underweight' AS CHAR)
    WHEN CAST(observed_weight/POWER(observed_length,2) AS DECIMAL) BETWEEN 15 AND 25 THEN CAST('Normal' AS CHAR)
    ELSE CAST('Overweight' AS CHAR)
END AS status_bmi
FROM crocodiles
ORDER BY id
```

# Datetime
Задача 39

Для каждого крокодила вывести, сколько дней прошло с момента наблюдения (observation_date)  до текущего момента. Отсортировать по observation_date.

```SQL
SELECT id, observation_date, DATEDIFF(NOW(),observation_date) AS days_since_observation
FROM crocodiles
ORDER BY observation_date
```

Задача 40

Вывести все наблюдения (observation_date), сделанные в апреле, мае или июне, и выведите дату в формате ДД.ММ.ГГГГ. Отсортировать по observation_date.

```SQL
SELECT id, DATE_FORMAT(observation_date, '%d.%m.%Y') AS formatted_date
FROM crocodiles
WHERE MONTH(observation_date) IN (4,5,6)
ORDER BY observation_date
```

Задача 41

Допустим, что каждое наблюдение (observation_date) крокодила проводится повторно через 3 недели после исходного.
Для каждого наблюдения выведите дату следующего осмотра. Отсортировать по observation_date.

```SQL
SELECT id, observation_date, DATE_ADD(observation_date, INTERVAL 3 WEEK) AS next_inspection_date
FROM crocodiles
ORDER BY observation_date
```

Задача 42

Найдите средний интервал между наблюдениями (observation_date) в днях для каждого региона.
В первой части запроса используйте СТЕ с оконной функцией LAG() для нахождения предыдущей даты каждого наблюдения. Чтобы получить средний интервал по региону, нужно взять среднее значение (AVG) разниц по дням и округлить его функцией ROUND(..., 2). Не забудьте исключить первые строки без предыдущей даты. Сгруппировать результат нужно по названию региона и отсортировать по среднему интервалу.

```SQL
WITH NEW_TABLE AS(
    SELECT dict_region.name AS region, DATEDIFF(observation_date, LAG(observation_date) OVER (PARTITION BY crocodiles.region_id)) AS days_between_obs
FROM crocodiles
JOIN dict_region ON dict_region.id = crocodiles.region_id),

NEW_TABLE2 AS(
SELECT * FROM NEW_TABLE
WHERE days_between_obs IS NOT NULL)

SELECT DISTINCT(region), ROUND(AVG(days_between_obs) OVER (PARTITION BY region), 2) AS avg_days_between_obs FROM NEW_TABLE2
ORDER BY avg_days_between_obs
```

# Транзакции
Задача 43

Сотрудник с id = 50 (Jerry Wheeler) переведён в архив и больше не может быть указан как наблюдатель.
Все его наблюдения нужно переназначить сотруднику с id = 15 (Matthew Lucas), который теперь отвечает за эти данные.
Напишите SQL-транзакцию, которая:
- Обновляет поле observer_id в таблице crocodiles:
- меняет 50 на 15 для всех соответствующих записей
- Выводит сообщение:
- "Переназначено X наблюдений от Jerry Wheeler к Matthew Lucas"
- Подтверждает транзакцию

```SQL
START TRANSACTION;
UPDATE crocodiles
SET observer_id = 15
WHERE observer_id = 50;
SELECT CONCAT('Переназначено ', ROW_COUNT(), ' наблюдений от Jerry Wheeler к Matthew Lucas');
COMMIT;
```

Задача 44

Из-за ошибки датчиков, все наблюдения за крокодилами с длиной меньше 0.1 метра считаются недостоверными и должны быть удалены.
Напишите SQL-транзакцию, которая:
- Удаляет все записи из таблицы crocodiles, где observed_length < 0.1
- Выводит сообщение в формате:
- "Удалено X недостоверных записей"
- Подтверждает транзакцию

```SQL
START TRANSACTION;
DELETE FROM crocodiles
WHERE observed_length < 0.1;
SELECT CONCAT('Удалено ', ROW_COUNT(), ' недостоверных записей');
COMMIT
```

# CRUD
Задача 45

Создайте таблицу employees_new для хранения информации о сотрудниках:

```SQL
CREATE TABLE employees_new(
    id INTEGER PRIMARY KEY,
    first_name VARCHAR(20),
    last_name VARCHAR(50),
    department VARCHAR(50),
    salary DECIMAL(10,2));
INSERT INTO employees_new(id, first_name, last_name, department, salary)
VALUES (1, 'Anna', 'Ivanova', 'IT', 95000),
(2, 'Petr', 'Sidorov', 'Sales', 85000),
(3, 'Olga', 'Smirnova', 'Interns', 30000),
(4, 'Nikita', 'Orlov', 'IT', 120000),
(5, 'Dmitry', 'Egorov', 'Finance', 70000)
```

Задача 46

Увеличьте зарплату на 10% всем сотрудникам из отдела 'IT'.

```SQL
UPDATE employees_new
SET salary = salary *1.1
WHERE department = 'IT'
```

Задача 47

Измените отдел сотрудников, чья зарплата превышает 100 000 на 'Management'

```SQL
UPDATE employees_new
SET department = 'Management'
WHERE salary > 100000
```

Задача 48

Удалите всех сотрудников, чья зарплата меньше 50000 или отдел 'Interns'.

```SQL
DELETE
FROM employees_new
WHERE salary<50000 OR department = 'Interns'
```

Задача 49

Добавьте нового сотрудника 'Irina', 'Smirnova', 'HR', 95000.
Уменьшите зарплату всех сотрудников из отдела 'Sales' на 5%.
Удалите всех сотрудников, у которых отдел 'Temporary'.
Обновите фамилию сотрудника с id = 1 на 'Petrov'.

```SQL
INSERT INTO employees_new(id, first_name, last_name, department, salary)
VALUES (6, 'Irina', 'Smirnova', 'HR', 95000);

UPDATE employees_new
SET salary = salary/1.05;

DELETE
FROM employees_new
WHERE department = 'Temporary';

UPDATE employees_new
SET last_name = 'Petrov'
WHERE id = 1;
```

Задача 50

Компания решила скорректировать зарплаты сотрудников в зависимости от их текущего уровня дохода.
Вам нужно обновить значения столбца salary с использованием конструкции CASE:
·        зарплата < 50 000 - увеличить на 20 %
·        от 50 000 до 99 999 - учивелить на 10 %
·        от 100 000 и выше - увеличить на 5 %

```SQL
UPDATE employees_new
    SET salary =
CASE 
    WHEN salary <50000 THEN salary = salary*1.2
    WHEN salary BETWEEN 50000 AND 99999 THEN salary = salary*1.1
    WHEN salary <50000 THEN salary = salary*1.05
END;
SELECT * FROM employees_new
ORDER BY salary
```







