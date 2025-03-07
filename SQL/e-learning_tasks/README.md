# SQL: Анализ данных для e-learning платформы

## Описание проекта
В этом проекте собраны аналитические задачи, связанные с работой образовательной платформы. Используя SQL, мы решаем прикладные бизнес-задачи, направленные на оптимизацию процессов и улучшение взаимодействия пользователей с платформой.

---

## Задачи
1. Идентификация усердных учеников  
   - Цель: определить количество студентов, которые правильно решили 20 или более задач за текущий месяц.  
   - Таблица: default.peas.  
   - Основные метрики:
     - Количество усердных студентов.
   - Задача на оптимизацию запроса для работы с большими объёмами данных.

2. Анализ эффективности нового экрана оплаты  
   - Цель: провести анализ воронки продаж в образовательной платформе.  
   - Таблицы: 
     - default.peas: данные о решённых задачах.
     - default.studs: информация о группах пользователей (контрольная или тестовая).
     - default.final_project_check: данные о покупках пользователей.  
   - Основные метрики:
     - ARPU (средний доход на пользователя).
     - ARPAU (средний доход на активного пользователя).
     - Конверсия пользователей в покупку (CR).
     - Конверсия активных пользователей в покупку.
     - Конверсия активных пользователей по математике в покупку курса по математике.

---

## Используемые данные
1. default.peas  
   - Поля:
     - st_id: ID ученика.
     - timest: время решения задачи.
     - correct: правильно ли решена задача (True/False).
     - subject: дисциплина задачи.

2. default.studs  
   - Поля:
     - st_id: ID ученика.
     - test_grp: группа пользователя (контрольная или тестовая).

3. default.final_project_check  
   - Поля:
     - st_id: ID ученика.
     - sale_time: время покупки.
     - money: стоимость покупки.
     - subject: дисциплина.
