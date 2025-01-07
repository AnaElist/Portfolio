# Проект: Анализ данных по объявлениям о сдаче жилья
Проект выполнен в рамках прохождения курсов аналитики данных Karpov.courses по дынным из открытого источника (https://insideairbnb.com/).  
Данный проект выполнен с использованием СУБД ClickHouse.

## Цель проекта

Цель выполнения проекта – отработка навыков работы с различными типами данных и их преобразования. 

## Описание данных

Проект анализирует объявления о сдаче жилья, предоставленные в таблице listings. Ключевые атрибуты данных:
- id: идентификатор жилья (объявления).
- room_type: тип жилья (например, 'Entire home/apt', 'Private room').
- price: цена за ночь.
- cleaning_fee: стоимость уборки.
- security_deposit: залог.
- neighbourhood_cleansed: район/округ города.
- square_feet: площадь жилья.
- latitude: широта.
- longitude: долгота.

## Задачи

1. На Airbnb есть две основные группы пользователей:
- Хозяева (хосты, hosts) – сдают жилье
- Путешественники (travelers) – снимают
Начнем с анализа характеристик хозяев в таблице listings!
Также обратите внимание, поле host_since содержит пустые значения.

Пользователи, сдающие квартиры на Airbnb, зарегистрировались в разное время. Кто-то – очень давно, а кто-то совсем недавно. Давайте проверим, в какой месяц и год зарегистрировалось наибольшее количество новых хостов.
```sql
SELECT 
  toStartOfMonth(toDateOrNull(host_since)) as h_month,
  count(distinct host_id)
FROM
  listings
GROUP BY h_month
ORDER BY host DESC
LIMIT 5 
```

2. Посмотрим на среднюю частоту ответа среди хозяев (f) и суперхозяев (t).
Значения частоты ответа хранятся как строки и включают значок %, который необходимо заменить на пустоту (''). После этого приведите столбец к нужному типу данных с помощью toInt32OrNull() и посчитайте среднюю частоту отклика в разбивке по тому, является ли хост суперхозяином или нет. В качестве ответа укажите наибольшее среднее.
 - host_response_rate – частота ответа
 - host_is_superhost – является ли суперхозяином
Важный момент: у каждого host_id есть только одно уникальное значение частоты ответа, а также одна единая отметка суперхозяина. Чтобы посчитать показатели честно, нужно использовать подзапрос и взять уникальные комбинации с помощью DISTINCT.

```sql
SELECT
  host_is_superhost,
  AVG(host_response_rate_int) as avg_host_response_rate
FROM
    (
    SELECT 
      DISTINCT host_id, 
      host_is_superhost, 
      toUint8OrNull(replaceAll(host_response_rate as, '%', '')
      ) as host_response_rate_int 
    FROM 
      listings
    ) 
GROUP BY host_is_superhost
LIMIT 5
```

3. Сгруппируйте данные из listings по хозяевам (host_id) и посчитайте, какую цену за ночь в среднем каждый из них устанавливает (у одного хоста может быть несколько объявлений). Идентификаторы сдаваемого жилья объедините в отдельный массив. Таблицу отсортируйте по убыванию средней цены и убыванию host_id (в таком порядке).

```sql
SELECT
  host_id,
  groupArray(id) as ids,
  AVG(toFloat32OrNull(replaceRegexpAll(price, '[$,]', ''))) as average_price
FROM
   listings
GROUP BY host_id
ORDER BY average_price DESC, host_id DESC
LIMIT 50
```
   
4. Немного усложним предыдущую задачу, и посчитаем разницу между максимальной и минимальной установленной ценой у каждого хозяина. В качестве ответа укажите идентификатор хоста, у которого разница оказалась наибольшей. 
- host_id – идентификатор хозяина
- id – идентификатор жилья
- price – цена за ночь в конкретном месте

```sql
SELECT
  host_id,
  MAX(toFloat32OrNull(
    replaceRegexpAll(price, '[$,]', '')
  ) -  
  MIN(toFloat32OrNull(
      replaceRegexpAll(price, '[$,]', '')
    ) as price_dif -- здесь берём разность между максимальной и минимальной стоимостью жилья, приведенной к числовому формату цены
 FROM
   listings
GROUP BY host_id
ORDER BY price_dif DESC
LIMIT 50
```

5. Теперь сгруппируйте данные по типу жилья и выведите средние значения цены за ночь, размера депозита и цены уборки. Обратите внимание на тип данных, наличие значка $ и запятых в больших суммах. Для какого типа жилья среднее значение залога наибольшее?
- room_type – тип сдаваемого жилья 
- price – цена за ночь
- security_deposit – залог за сохранность имущества
- cleaning_fee – плата за уборку

```sql
SELECT
  room_type,
  AVG(toFloat32OrNull(replaceRegexpAll(price, '[$,]', ''))) as avg_price,
  AVG(toFloat32OrNull(replaceRegexpAll(cleaning_fee, '[$,]', '')))  as avg_cleaning_fee,
  AVG(toFloat32OrNull(replaceRegexpAll(security_deposit, '[$,]', '')))  as avg_security_deposit
FROM
   listings
GROUP BY room_type
LIMIT 5
```

6.  В каких частях города средняя стоимость за ночь является наиболее низкой? 
Сгруппируйте данные по neighbourhood_cleansed и посчитайте среднюю цену за ночь в каждом районе. В качестве ответа введите название места, где средняя стоимость за ночь ниже всего.
- price – цена за ночь
- neighbourhood_cleansed – район/округ города

```sql
SELECT
  neighbourhood_cleansed,
  AVG(toFloat32OrNull(replaceRegexpAll(price, '[$,]', ''))) as avg_price
FROM
   listings
GROUP BY neighbourhood_cleansed
ORDER BY avg_price
LIMIT 50
```

7.В каких районах Берлина средняя площадь жилья, которое сдаётся целиком, является наибольшей? Отсортируйте по среднему и выберите топ-3. 
- neighbourhood_cleansed – район
- square_feet – площадь
- room_type – тип (нужный – 'Entire home/apt')

```sql
SELECT
  neighbourhood_cleansed,
  AVG(toFloat32OrNull(square_feet)) as avg_square_feet
FROM
   listings
WHERE room_type = 'Entire home/apt'
GROUP BY neighbourhood_cleansed
ORDER BY avg_square_feet DESC
LIMIT 50
```

8. Напоследок давайте посмотрим, какая из представленных комнат расположена ближе всего к центру города. В качестве ответа укажите id объявления.
- id – идентификатор жилья (объявления)
- room_type – тип жилья ('Private room')
- latitude – широта
- longitude – долгота
- 52.5200 с.ш., 13.4050 в.д – координаты центра Берлина

```sql
SELECT
  id,
  toFloat32OrNull(longitude) as longitude_float,
  toFloat32OrNull(latitude) as latitude_float,
  geoDistance(13.4050, 52.5200, longitude_float, latitude_float) as distance
FROM
   listings
WHERE room_type = 'Private room'
ORDER BY distance 
LIMIT 20
```
