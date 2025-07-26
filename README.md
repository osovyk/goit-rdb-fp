# Опис фінального проєкту

1. Завантажте дані:
* Створіть схему pandemic у базі даних за допомогою SQL-команди.
```
CREATE SCHEMA pandemic;
```
* Оберіть її як схему за замовчуванням за допомогою SQL-команди.
```
USE pandemic;
```
* Імпортуйте дані за допомогою Import wizard так, як ви вже робили це у темі 3.
* Продивіться дані, щоб бути у контексті.
> 💡 Як бачите, атрибути Entity та Code постійно повторюються. Позбудьтеся цього за допомогою нормалізації даних.

2. Нормалізуйте таблицю infectious_cases до 3ї нормальної форми. Збережіть у цій же схемі дві таблиці з нормалізованими даними.
```
-- 2. Нормалізація до 3NF
-- Таблиця для унікальних Entity та Code
CREATE TABLE entities (
    entity_id INT AUTO_INCREMENT PRIMARY KEY,
    entity_name TEXT NOT NULL,
    code TEXT,
    UNIQUE (entity_name(100), code(10))
);



-- Таблиця для даних про захворювання
CREATE TABLE disease_cases (
    case_id INT AUTO_INCREMENT PRIMARY KEY,
    entity_id INT,
    year INT,
    number_yaws INT,
    polio_cases INT,
    cases_guinea_worm INT,
    number_rabies DOUBLE,
    number_malaria DOUBLE,
    number_hiv DOUBLE,
    number_tuberculosis DOUBLE,
    number_smallpox INT,
    number_cholera_cases INT,
    FOREIGN KEY (entity_id) REFERENCES entities(entity_id)
);
```
```
-- Заповнення нормалізованої таблиці disease_cases
INSERT INTO disease_cases (
    entity_id, 
    year, 
    number_yaws, 
    polio_cases, 
    cases_guinea_worm, 
    number_rabies, 
    number_malaria, 
    number_hiv, 
    number_tuberculosis, 
    number_smallpox, 
    number_cholera_cases
)
SELECT 
    e.entity_id, 
    ic.Year, 
    NULLIF(ic.Number_yaws, '') AS number_yaws,
    NULLIF(ic.polio_cases, '') AS polio_cases,
    NULLIF(ic.cases_guinea_worm, '') AS cases_guinea_worm,
    NULLIF(ic.Number_rabies, '') AS number_rabies,
    NULLIF(ic.Number_malaria, '') AS number_malaria,
    NULLIF(ic.Number_hiv, '') AS number_hiv,
    NULLIF(ic.Number_tuberculosis, '') AS number_tuberculosis,
    NULLIF(ic.Number_smallpox, '') AS number_smallpox,
    NULLIF(ic.Number_cholera_cases, '') AS number_cholera_cases
FROM infectious_cases ic
JOIN entities e ON ic.Entity = e.entity_name AND (ic.Code = e.code OR (ic.Code IS NULL AND e.code IS NULL));

-- Підрахунок кількості записів у вихідній таблиці
SELECT COUNT(*) AS record_count FROM infectious_cases;
```
> 💡 Виконайте запит SELECT COUNT(*) FROM infectious_cases , щоб ментор міг зрозуміти, скільки записів ви завантажили у базу даних із файла.

3. Проаналізуйте дані:
* Для кожної унікальної комбінації Entity та Code або їх id порахуйте середнє, мінімальне, максимальне значення та суму для атрибута Number_rabies.
> 💡 Врахуйте, що атрибут Number_rabies може містити порожні значення ‘’ — вам попередньо необхідно їх відфільтрувати.
* Результат відсортуйте за порахованим середнім значенням у порядку спадання.
* Оберіть тільки 10 рядків для виведення на екран.
```
SELECT 
    e.entity_name,
    e.code,
    AVG(dc.number_rabies) AS avg_rabies,
    MIN(dc.number_rabies) AS min_rabies,
    MAX(dc.number_rabies) AS max_rabies,
    SUM(dc.number_rabies) AS sum_rabies
FROM disease_cases dc
JOIN entities e ON dc.entity_id = e.entity_id
WHERE dc.number_rabies IS NOT NULL
GROUP BY e.entity_id, e.entity_name, e.code
ORDER BY avg_rabies DESC
LIMIT 10;
```

4. Побудуйте колонку різниці в роках.
Для оригінальної або нормованої таблиці для колонки Year побудуйте з використанням вбудованих SQL-функцій:
* атрибут, що створює дату першого січня відповідного року,
> 💡 Наприклад, якщо атрибут містить значення ’1996’, то значення нового атрибута має бути ‘1996-01-01’.
* атрибут, що дорівнює поточній даті,
* атрибут, що дорівнює різниці в роках двох вищезгаданих колонок.
> 💡 Перераховувати всі інші атрибути, такі як Number_malaria, не потрібно.
> 👉🏼 Для пошуку необхідних вбудованих функцій вам може знадобитися матеріал до теми 7.
```
SELECT
  year,
  DATE(CONCAT(year, '-01-01'))          AS first_jan_date,
  TIMESTAMPDIFF(YEAR,
                DATE(CONCAT(year, '-01-01')),
                CURDATE())                  AS years_since_year
FROM infectious_cases
```

5. Побудуйте власну функцію.
Створіть і використайте функцію, що будує такий же атрибут, як і в попередньому завданні: функція має приймати на вхід значення року, а повертати різницю в роках між поточною датою та датою, створеною з атрибута року (1996 рік → ‘1996-01-01’).
> 💡 Якщо ви не виконали попереднє завдання, то можете побудувати іншу функцію — функцію, що рахує кількість захворювань за певний період. Для цього треба поділити кількість захворювань на рік на певне число: 12 — для отримання середньої кількості захворювань на місяць, 4 — на квартал або 2 — на півріччя. Таким чином, функція буде приймати два параметри: кількість захворювань на рік та довільний дільник. Ви також маєте використати її — запустити на даних. Оскільки не всі рядки містять число захворювань, вам необхідно буде відсіяти ті, що не мають чисельного значення (≠ ‘’).
```
DROP FUNCTION IF EXISTS CalculateYearDifference;
DELIMITER //
CREATE FUNCTION CalculateYearDifference(year_input YEAR)
RETURNS INT DETERMINISTIC
BEGIN
  RETURN TIMESTAMPDIFF(
	YEAR,
	DATE(CONCAT(year_input, '-01-01')),
	CURDATE()
	);
END //

DELIMITER ;

SELECT
  year,
  CalculateYearDifference(year) AS year_difference
FROM infectious_cases
```
