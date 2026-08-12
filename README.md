# SQL
# В этой ветке выкладываю тренировочные SQL запросы


## CTE, множественный и рекурсиный CTE
###Задача 1

<img width="840" height="172" alt="image" src="https://github.com/user-attachments/assets/388fa7ec-0a60-40dd-9d0a-3ec49177fc13" />


###Задача 2
С помощью CTE найдите среднюю длину крокодилов по каждому региону, а затем выведите только те регионы, где средняя длина больше 3 метров.
Решение
WITH SrDlina AS (
    SELECT dict_region.name AS region, AVG (observed_length) AS avg_length
    FROM crocodiles
    JOIN dict_region ON crocodiles.region_id = dict_region.id
    GROUP BY dict_region.name
    HAVING AVG(observed_length) > 3
    ORDER BY AVG(observed_length) DESC)
SELECT * FROM SrDlina

###Задача 3
С помощью CTE найдите среднюю длину крокодилов по каждому региону, а затем выведите только те регионы, где средняя длина больше 3 метров.
Решение
WITH SrDlina AS (
    SELECT dict_region.name AS region, AVG (observed_length) AS avg_length
    FROM crocodiles
    JOIN dict_region ON crocodiles.region_id = dict_region.id
    GROUP BY crocodiles.region_id
    HAVING AVG (observed_length) > 3
    ORDER BY dict_region.name)
SELECT * FROM SrDlina

###Задача 4
4.1. На первом этапе добавьте каждому наблюдению номер строки (ROW_NUMBER) внутри региона (по дате наблюдения).
4.2. На втором этапе выберите только первые наблюдения по каждому региону.
Решение
WITH Otvet AS (
    SELECT crocodiles.id, dict_region.name AS region, dict_common_name.name AS common_name, observation_date, ROW_NUMBER() OVER (PARTITION BY crocodiles.region_id ORDER BY observation_date) AS obs_rank
    FROM crocodiles
    JOIN dict_region ON crocodiles.region_id = dict_region.id
    JOIN dict_common_name ON crocodiles.common_name_id = dict_common_name.id)
SELECT * FROM Otvet
WHERE obs_rank = 1
ORDER BY observation_date DESC

###Задача 5
С помощью конструкции WITH и оконной функции определите самого тяжёлого крокодила в каждом виде.
Решение
WITH tzCroc AS (
    SELECT crocodiles.id, dict_common_name.name AS common_name, observed_weight, MAX(crocodiles.observed_weight) OVER (PARTITION BY crocodiles.common_name_id) AS max_weight
    FROM crocodiles
    JOIN dict_common_name ON crocodiles.common_name_id = dict_common_name.id)
SELECT * FROM tzCroc
HAVING observed_weight=max_weight
ORDER BY observed_weight DESC, common_name

###Задача 6
6.1. В первом CTE (large_crocs) выберите всех крокодилов длиной более 4 метров.
6.2. Во втором CTE (species_count) посчитайте количество таких крупных крокодилов для каждого вида.
6.3. В основном запросе выведите виды, у которых есть хотя бы один крупный крокодил, и их количество.
6.4. Отсортируйте по полю large_countпо убыванию. 
Решение
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

###Задача 7
7.1. В первом CTE (region_stats) для каждого региона посчитайте общее количество наблюдений и средний вес крокодилов.
7.2. Во втором CTE (filtered_regions) отфильтруйте результаты первого CTE, оставив только регионы со средним весом более 500 кг.
7.3. В основном запросе присоедините отфильтрованный список регионов к исходной таблице crocodiles, чтобы вывести всех крокодилов из этих "тяжелых" регионов.
7.4. Отсортировать по умолчанию по полям region_name и id.
Решение
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

###Задача 8
Создайте простой список, показывающий статус сохранения вида каждого крокодила и фамилию сотрудника, который его наблюдал. Используйте два CTE: один для статусов сохранения, другой для информации о сотрудниках. Выведите только 7 записей.
Решение
WITH CS AS(
    SELECT dict_conservation_status.name AS conservation_status, observer_id FROM crocodiles
    JOIN dict_conservation_status ON crocodiles.conservation_status_id = dict_conservation_status.id),
OLN AS (
    SELECT CS.conservation_status, employee.last_name AS observer_last_name FROM CS
    JOIN employee ON CS.observer_id = employee.id)
SELECT * FROM OLN
LIMIT 7;

###Задача 9
Покажите список, в котором для каждого крокодила указана страна, где он был найден, и тип среды обитания. Используйте два CTE: один для регионов, другой для мест обитания. Выведите только 10 первых записей.
Решение
WITH REG AS (
    SELECT dict_region.name AS region, crocodiles.habitat_id FROM crocodiles
    JOIN dict_region ON crocodiles.region_id = dict_region.id),
HAB AS (
    SELECT region, dict_habitats.name AS habitat FROM REG
    JOIN dict_habitats ON REG.habitat_id = dict_habitats.id)     
SELECT * FROM HAB
LIMIT 10

###Задача 10
Посчитайте, сколько крокодилов каждого возраста и пола было зафиксировано. Используйте два CTE: один для расшифровки возраста, другой для расшифровки пола. Сгруппируйте результаты по возрасту и полу.
Решение
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

###Задача 11
Создайте иерархию весовых категорий:
    Легкие: < 200 кг
    Средние: 200-400 кг
    Тяжелые: > 400 кг
Выведите количество крокодилов в каждой категории.
Решение
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

## Оконные функции
###Задача 12


