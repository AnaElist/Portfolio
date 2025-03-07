![image](https://github.com/user-attachments/assets/18cd5981-ff62-4513-81ed-46b63ba533b8)

**Задание 1.**
Найдите сумму транзакций каждого пациента за 2023 год. Выведите id пациента, сумму всех его транзакций.
```sql
SELECT 
	DIM_PATIENTS_ID,
	SUM(REVENUE) AS REVENUE_PER_PATIENT
FROM
	FACT_COMMERCIAL_STATS
WHERE
	BILL_DATE >= '2023-01-01' AND BILL_DATE <= '2023-12-31'
GROUP BY
	DIM_PATIENTS_ID;
```
**Задание 2.**
Для каждого врача в филиале Отделение "Южное" выведите сумму выручки и количество приемов с 1 января 2024 по услуге Акция: Первичный прием. Знакомство с врачом. Выведите ФИО врача, выручку, количество приемов в порядке убывания выручки.
```sql
SELECT 
	DE.FULL_NAME AS DOCTOR_FULL_NAME,
	SUM(FCS.REVENUE) AS REVENUE_PER_DOCTOR,
	COUNT(FCS.DIM_PATIENTS_ID) AS APPOINTMENT_COUNT
FROM
	FACT_COMMERCIAL_STATS AS FCS
INNER JOIN 
	DIM_EMPLOYEE AS DE ON FCS.DIM_EMPLOYEE_ID = DE.DIM_EMPLOYEE_ID 
INNER JOIN 
	DIM_ORG_STRUCTURE AS DOS ON FCS.DIM_ORG_STRUCTURE_ID = DOS.DIM_ORG_STRUCTURE_ID
INNER JOIN
	 DIM_SERV AS DS ON FCS.DIM_SERV_ID = DS.DIM_SERV_ID
WHERE
	DOS.FILIAL = 'Отделение "Южное"'
	AND FCS.BILL_DATE >= '2024-01-01'
	AND DS.SERVICE_NAME = 'Акция: Первичный прием. Знакомство с врачом.'
GROUP BY
	DE.FULL_NAME
ORDER BY
	REVENUE_PER_DOCTOR DESC;
```

**Задание 3.**
Найдите пациентов, которые год не были на приеме у стоматолога в филиалах Отделение "Московское" и Отделение "Северное". Выведите также для каждого пациента дату последнего приема у стоматолога. 
```sql
SELECT
	FCS.DIM_PATIENTS_ID,
	MAX(BILL_DATE) AS LAST_APPOINTMENT_DATE
FROM
	FACT_COMMERCIAL_STATS AS FCS
INNER JOIN
	DIM_ORG_STRUCTURE AS DOS ON FCS.DIM_ORG_STRUCTURE_ID = DOS.DIM_ORG_STRUCTURE_ID
INNER JOIN
	DIM_SERV AS DS ON DS.DIM_SERV_ID = FCS.DIM_SERV_ID
WHERE
	DOS.FILIAL IN ('Отделение "Московское"', 'Отделение "Северное"')
	AND SERVICE_NAME IN ('Первичный приём стоматолога', 'Повторный приём стоматолога')
	-- Или если нет точного списка услуг, можно попробовать  DS.SERVICE_NAME LIKE '%стоматолог%'
GROUP BY
	FCS.DIM_PATIENTS_ID
HAVING
	MAX(FCS.BILL_DATE) < CURRENT_DATE - INTERVAL '1 year' ;
-- правильное написание запроса с функциями времени будет зависеть от диалекта, выше использую postgres
```
**Задание 4**

a)	Филиал Отделение "Обводный Канал". 

- Найдите пациентов, которые год не были на приеме у гинеколога (вне зависимости, когда они делали УЗИ).
- Найдите пациентов, которые не делали УЗИ больше года (вне зависимости, когда они были у гинеколога). Интересуют только УЗИ молочных желез, УЗИ щитовидной железы, УЗИ брюшной полости.
- Выведите общий список уникальных id пациентов.

```sql
SELECT 
	DIM_PATIENTS_ID
FROM (
	-- Пациенты, не посещавшие гинеколога больше года
	SELECT 
		FCS.DIM_PATIENTS_ID
	FROM 
		FACT_COMMERCIAL_STATS AS FCS
	INNER JOIN
		DIM_ORG_STRUCTURE AS DOS ON FCS.DIM_ORG_STRUCTURE_ID = DOS.DIM_ORG_STRUCTURE_ID
	INNER JOIN
		DIM_SERV AS DS ON DS.DIM_SERV_ID = FCS.DIM_SERV_ID
	WHERE
		DOS.FILIAL = 'Отделение "Обводный Канал"'
		AND DS.SERVICE_NAME IN ('Первичный приём гинеколога', 'Повторный приём гинеколога')
	GROUP BY
		FCS.DIM_PATIENTS_ID
	HAVING 
		MAX(FCS.BILL_DATE) < CURRENT_DATE - INTERVAL '1 year'
	
	UNION

	-- Пациенты, не делавшие УЗИ больше года
	SELECT 
		FCS.DIM_PATIENTS_ID
	FROM FACT_COMMERCIAL_STATS AS FCS
	INNER JOIN
		DIM_ORG_STRUCTURE AS DOS ON FCS.DIM_ORG_STRUCTURE_ID = DOS.DIM_ORG_STRUCTURE_ID
	INNER JOIN 
		DIM_SERV AS DS ON DS.DIM_SERV_ID = FCS.DIM_SERV_ID
	WHERE
		DOS.FILIAL = 'Отделение "Обводный Канал"'
		AND DS.SERVICE_NAME IN ('УЗИ молочных желез', 'УЗИ щитовидной железы', 'УЗИ брюшной полости')
	GROUP BY
		FCS.DIM_PATIENTS_ID
	HAVING
		MAX(FCS.BILL_DATE) < CURRENT_DATE - INTERVAL '1 year'
) AS patients_no_gyn_no_us;
```
b)	*** Для пациентов из пункта a) выведите точку входа в клинику. Точка входа в данном случае – это дата первой транзакции, ФИО врача и название филиала в этой транзакции.

```sql
WITH patients_list AS (
    -- Пациенты, не посещавшие гинеколога или не делавшие УЗИ больше года
    SELECT DIM_PATIENTS_ID
    FROM (
        SELECT 
            FCS.DIM_PATIENTS_ID
        FROM FACT_COMMERCIAL_STATS AS FCS
        INNER JOIN DIM_ORG_STRUCTURE AS DOS ON FCS.DIM_ORG_STRUCTURE_ID = DOS.DIM_ORG_STRUCTURE_ID
        INNER JOIN DIM_SERV AS DS ON DS.DIM_SERV_ID = FCS.DIM_SERV_ID
        WHERE DOS.FILIAL = 'Отделение "Обводный Канал"'
          AND DS.SERVICE_NAME IN ('Первичный приём гинеколога', 'Повторный приём гинеколога')
        GROUP BY FCS.DIM_PATIENTS_ID
        HAVING MAX(FCS.BILL_DATE) < CURRENT_DATE - INTERVAL '1 year'

        UNION

        SELECT 
            FCS.DIM_PATIENTS_ID
        FROM FACT_COMMERCIAL_STATS AS FCS
        INNER JOIN DIM_ORG_STRUCTURE AS DOS ON FCS.DIM_ORG_STRUCTURE_ID = DOS.DIM_ORG_STRUCTURE_ID
        INNER JOIN DIM_SERV AS DS ON DS.DIM_SERV_ID = FCS.DIM_SERV_ID
        WHERE DOS.FILIAL = 'Отделение "Обводный Канал"'
          AND DS.SERVICE_NAME IN ('УЗИ молочных желез', 'УЗИ щитовидной железы', 'УЗИ брюшной полости')
        GROUP BY FCS.DIM_PATIENTS_ID
        HAVING MAX(FCS.BILL_DATE) < CURRENT_DATE - INTERVAL '1 year'
    ) AS patients_no_gyn_no_us
), 

first_visits AS (
    -- Назначаем номер строки для первой транзакции каждого пациента
    SELECT 
        FCS.DIM_PATIENTS_ID,
        FCS.BILL_DATE AS FIRST_VISIT_DATE,
        DE.FULL_NAME AS DOCTOR_FULL_NAME,
        DOS.FILIAL,
        ROW_NUMBER() OVER (PARTITION BY FCS.DIM_PATIENTS_ID ORDER BY FCS.BILL_DATE ASC) AS rn
    FROM FACT_COMMERCIAL_STATS AS FCS
    INNER JOIN DIM_EMPLOYEE AS DE ON FCS.DIM_EMPLOYEE_ID = DE.DIM_EMPLOYEE_ID
    INNER JOIN DIM_ORG_STRUCTURE AS DOS ON FCS.DIM_ORG_STRUCTURE_ID = DOS.DIM_ORG_STRUCTURE_ID
    INNER JOIN patients_list AS pl ON FCS.DIM_PATIENTS_ID = pl.DIM_PATIENTS_ID)

-- Оставляем только первое посещение
SELECT 
    fv.DIM_PATIENTS_ID,
    fv.FIRST_VISIT_DATE,
    fv.DOCTOR_FULL_NAME,
    fv.FILIAL
FROM first_visits AS fv
WHERE rn = 1;
```
