# Анализ данных мобильного приложения

## Описание данных

### 1. Таблица `installs`
- **DeviceID** — идентификатор устройства, на которое было установлено приложение.
- **InstallationDate** — дата установки приложения.
- **InstallCost** — цена установки приложения в рублях.
- **Platform** — платформа, на которой было установлено приложение (iOS/Android).
- **Source** — источник установки приложения (магазин приложения/рекламная система/переход с сайта).

### 2. Таблица `events`
- **DeviceID** — идентификатор устройства, на котором используется приложение.
- **AppPlatform** — платформа, на которой используется приложение (iOS/Android).
- **EventDate** — дата, за которую собрана статистика.
- **events** — количество просмотров всех товаров за этот день у этого DeviceID.

### 3. Таблица `checks`
- **UserID** — идентификатор пользователя.
- **Rub** — суммарный чек пользователя на дату.
- **BuyDate** — дата, за которую собрана статистика.

### 4. Таблица `devices`
- **DeviceID** — идентификатор устройства.
- **UserID** — идентификатор пользователя.


## Особенности данных

- Для просмотра товаров в приложении авторизация необязательна, поэтому в `installs` и `events` фиксируется только `DeviceID`.
- Покупки возможны только после авторизации, поэтому в таблице `checks` фиксируется только `UserID`.
- Таблица `devices` связывает `DeviceID` и `UserID`.

---

## Решение задач

### 1. 
У пользователя может быть два идентификатора – UserID и DeviceID. В таблице checks есть только UserID, в остальных – только DeviceID. Во вспомогательной таблице devices есть и UserID, и DeviceID. 
Давайте с помощью JOIN дополним таблицу events (left) данными о UserID пользователей из таблицы devices (right). Для некоторых DeviceID не будет пары UserID из таблицы devices – подумайте, какой вид JOIN подойдет, чтобы не потерять те строки, где DeviceID есть в events, но нет в devices.
Укажите UserID из первой строки результирующей таблицы, используя сортировку по убыванию по полю DeviceID.

```sql
SELECT 
  * 
FROM 
  events as a 
LEFT JOIN 
    devices as b ON a.DeviceID = b.DeviceID 
LIMIT 
  5 
```

### 2.
Давайте проверим, пользователи пришедшие из какого источника совершили наибольшее число покупок. В качестве ответа выберите название Source, юзеры которого совершили больше всего покупок.
Hint: Для этого используйте UserID, DeviceID и Source из соответствующих таблиц. Считать уникальные значения здесь не нужно. Также заметьте, что покупки со стоимостью 0 рублей всё ещё считаются покупками.

```sql
SELECT 
  i.Source as iSource,
  COUNT(*) as sell_count
FROM 
  installs as i
INNER JOIN
  devices as d ON d.DeviceID = i.DeviceID
INNER JOIN
  checks as c ON c.UserID =  d.UserID
GROUP BY iSource
ORDER BY sell_count DESC
LIMIT 
  5 
```

### 3.
Теперь выясним, сколько всего уникальных юзеров совершили покупки в нашем приложении. 
Объедините нужные таблицы, посчитайте число уникальных UserID для каждого источника (Source), и в качестве ответа укажите число пользователей, пришедших из Source_7.
 Hint: checks – покупки, devices – соответствие, installs – информация об источнике.

```sql
SELECT 
  i.Source as iSource,
  uniqExact(d.UserID)
FROM 
  installs as i
INNER JOIN
  devices as d ON d.DeviceID = i.DeviceID
INNER JOIN
  checks as c ON c.UserID =  d.UserID
GROUP BY iSource
```



### 4. 
Самое время посмотреть на общую выручку, а также минимальный, максимальный и средний чек. Рассчитайте нужные показатели.
   
```sql
SELECT 
  i.Source as iSource,
  MIN(Rub) as min_check,
  MAX(Rub) as max_check,
  AVG(Rub) as avg_check,
  SUM(Rub) as sum_check
FROM 
  installs as i
INNER JOIN
  devices as d ON d.DeviceID = i.DeviceID
INNER JOIN
  checks as c ON c.UserID =  d.UserID
GROUP BY iSource
```


### 5. 
Выведите идентификаторы устройств пользователей, которые совершили как минимум одну покупку за последний месяц (октябрь 2019). Используйте сортировку по возрастанию DeviceID и укажите минимальное значение.
Hint: для извлечения месяца из даты можно использовать toMonth() или  toStartOfMonth(), предварительно приведя BuyDate к типу date.

```sql
SELECT 
  *
FROM 
  checks as c
INNER JOIN
  devices as d ON d.UserID =  c.UserID
WHERE
  toStartOfMonth(CAST(c.BuyDate as Date)) = Date '2019-10-01'
ORDER BY d.DeviceID
LIMIT 1
```

### 6. 
Проверим, сколько товаров (events) в среднем просматривают пользователи с разных платформ (Platform), и пришедшие из разных источников  (Source). Для этого объедините таблицы events и installs, и посчитайте, сколько просмотров в среднем приходится на каждую пару платформа-канал привлечения.
Отсортируйте полученную табличку по убыванию среднего числа просмотров. В качестве ответа укажите платформу и источник, пользователи которого в среднем просматривали товары бóльшее число раз.

```sql
SELECT 
  i.Source as iSource,
  e.AppPlatform as AppPlatform,
  AVG(events) as avg_events
FROM 
  events as e
INNER JOIN
  installs as i ON i.DeviceID = e.DeviceID
GROUP BY iSource, AppPlatform
ORDER BY avg_events DESC
```

### 7. 
Давайте посчитаем число уникальных DeviceID в инсталлах, для которых присутствуют просмотры в таблице events с разбивкой по платформам (поле Platform). Для этого можно отобрать все строки только из таблицы installs, для которых нашлось соответствие в таблице events. 
Внимание! "Нашлось 0 просмотров" тоже считается как "нашлись просмотры". Главное, чтобы не было именно пропуска.

```sql
SELECT 
  e.AppPlatform as AppPlatform,
  uniqExact(i.DeviceID)
FROM 
  installs as i
LEFT SEMI JOIN
  events as e ON e.DeviceID = i.DeviceID
GROUP BY AppPlatform
```

### 8.
Давайте теперь посчитаем конверсию из инсталла в просмотр с разбивкой по платформе инсталла – в данном случае это доля DeviceID, для которых есть просмотры, от всех DeviceID в инсталлах. 
Для этого нужно объединить таблицы installs и events так, чтобы получить все DeviceID инсталлов и соответствующие им DeviceID из events, посчитать число уникальных DeviceID инсталлов (1) и соответствующих DeviceID из events (2) и вычислить долю (2) от (1). 

```sql
SELECT 
   uniqExact(e.DeviceID)/uniqExact(i.DeviceID)
FROM 
  installs as i
LEFT JOIN
  events as e ON e.DeviceID = i.DeviceID
```

### 9. 
Представим, что в логирование DeviceID в событиях закралась ошибка - часть ID была записана в базу некорректно. Это привело к тому, что в таблице с событиями появились DeviceID, для которых нет инсталлов. Нам надо отобрать примеры DeviceID из таблицы event, которых нет в таблице installs, чтобы отправить их команде разработчиков на исправление. 
Выведите 10 уникальных DeviceID, которые присутствуют в таблице events, но отсутствуют в installs, отсортировав их в порядке убывания. 

```sql
SELECT 
   DISTINCT DeviceID
FROM 
  events as e
LEFT ANTI JOIN
  installs as i ON i.DeviceID = e.DeviceID
ORDER BY DeviceID DESC
LIMIT 10
```
