# В этой ветке выкладываю тренировочные SQL запросы


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

4.1. На первом этапе добавьте каждому наблюдению номер строки (ROW_NUMBER) внутри региона (по дате наблюдения).
4.2. На втором этапе выберите только первые наблюдения по каждому региону.

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

6.1. В первом CTE (large_crocs) выберите всех крокодилов длиной более 4 метров.
6.2. Во втором CTE (species_count) посчитайте количество таких крупных крокодилов для каждого вида.
6.3. В основном запросе выведите виды, у которых есть хотя бы один крупный крокодил, и их количество.
6.4. Отсортируйте по полю large_countпо убыванию. 

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
7.1. В первом CTE (region_stats) для каждого региона посчитайте общее количество наблюдений и средний вес крокодилов.
7.2. Во втором CTE (filtered_regions) отфильтруйте результаты первого CTE, оставив только регионы со средним весом более 500 кг.
7.3. В основном запросе присоедините отфильтрованный список регионов к исходной таблице crocodiles, чтобы вывести всех крокодилов из этих "тяжелых" регионов.
7.4. Отсортировать по умолчанию по полям region_name и id.

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
Посчитайте, сколько крокодилов каждого возраста и пола было зафиксировано. Используйте два CTE: один для расшифровки возраста, другой для расшифровки пола. Сгруппируйте результаты по возрасту и полу.

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
    Легкие: < 200 кг
    Средние: 200-400 кг
    Тяжелые: > 400 кг
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
Задача 12

Выведите для каждой записи из таблицы crocodiles её id, название вида крокодила и его вес. Добавьте колонку с суммарным весом всех крокодилов этого вида.

```SQL
SELECT crocodiles.id, dict_common_name.name, observed_weight, SUM(observed_weight) OVER (PARTITION BY common_name_id) AS total_weight
FROM crocodiles
JOIN dict_common_name ON dict_common_name.id = crocodiles.common_name_id
ORDER BY name, id
```

Задача 13

Выведите для каждой записи из таблицы crocodiles её id, название региона, дату наблюдения. Добавьте колонку с самой ранней датой наблюдения в этом регионе.

```SQL
SELECT crocodiles.id, dict_region.name AS region, observation_date, MIN(observation_date) OVER (PARTITION BY crocodiles.region_id) first_observation
FROM crocodiles
JOIN dict_region ON dict_region.id = crocodiles.region_id
ORDER BY observation_date DESC
```

Задача 14

Ученые хотят понять, насколько вес каждого отдельного крокодила отличается от среднего веса для его вида. Это поможет выявить особей с аномально низким или высоким весом.

```SQL
SELECT crocodiles.id, crocodiles.common_name_id, observed_weight, AVG(observed_weight) OVER (PARTITION BY common_name_id) AS avg_weight_forspecies, observed_weight-AVG(observed_weight) OVER (PARTITION BY common_name_id) AS weight_difference_from_avf
FROM crocodiles
ORDER BY common_name_id, id
```

Задача 15

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

Задача 16

Исследователям нужно отслеживать общий вес всех крокодилов, наблюдавшихся нарастающим итогом по мере проведения наблюдений.

```SQL
SELECT 
    observation_date, 
    observed_weight, 
    SUM(observed_weight) OVER (ORDER BY observation_date) AS cumulative_total_weight
FROM crocodiles
ORDER BY observation_date
```

Задача 17

Выведите для каждой записи из таблицы crocodiles её id, название вида и дату наблюдения. Добавьте порядковый номер наблюдения по дате (от самого раннего к самому позднему).

```SQL
SELECT crocodiles.id, dict_common_name.name, observation_date, ROW_NUMBER() OVER (ORDER BY observation_date) AS obs_number
FROM crocodiles
JOIN dict_common_name ON dict_common_name.id = crocodiles.id
ORDER BY obs_number
```

Задача 18

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

Задача 19

Выведите для каждой записи из таблицы crocodiles её id, название региона, дату наблюдения и вид крокодила. Добавьте колонку с датой предыдущего наблюдения в этом же регионе (по времени).

```SQL
SELECT crocodiles.id AS id, dict_region.name AS region, dict_common_name.name AS common_name, observation_date AS observation_date, LAG(observation_date) OVER (PARTITION BY crocodiles.region_id ORDER BY observation_date) AS prev_observation_date
FROM crocodiles
JOIN dict_common_name ON crocodiles.common_name_id = dict_common_name.id
JOIN dict_region ON dict_region.id = crocodiles.region_id
ORDER BY crocodiles.id
```

Задача 20

Выведите для каждой записи её id, название региона, дату наблюдения и длину крокодила. Добавьте колонку с длиной следующего крокодила, зафиксированного в том же регионе (по времени).

```SQL
SELECT crocodiles.id, dict_region.name AS region, observation_date, observed_length, LEAD(observed_length) OVER (PARTITION BY crocodiles.region_id ORDER BY observation_date) AS next_length
FROM crocodiles
JOIN dict_region ON dict_region.id = crocodiles.region_id
ORDER BY crocodiles.id
```

Задача 21

Для планирования программ сохранения нужно определить трех самых тяжелых крокодилов в каждом регионе наблюдения.
Что нужно сделать:Составить рейтинг крокодилов по весу внутри каждого региона и выбрать только первых трех.

```SQL
SELECT region_id, observed_weight, weight_rank
FROM (SELECT crocodiles.region_id AS region_id, observed_weight, RANK() OVER (PARTITION BY region_id ORDER BY observed_weight DESC) AS weight_rank FROM crocodiles) AS NewTable
WHERE NewTable.weight_rank <= 3
ORDER BY region_id, weight_rank
```

Задача 22
```SQL
```

Задача 23
```SQL
```

Задача 24
```SQL
```

Задача 25
```SQL
```

Задача 26
```SQL
```

Задача 27
```SQL
```

Задача 28
```SQL
```

Задача 29
```SQL
```
















