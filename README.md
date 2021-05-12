# 1. Выбор темы
Аналог Instagram, направленный на российскую аудиторию. MVP - личный профиль, подписки, лента, комментарии и лайки

# 2. Расчет нагрузки

## 2.1. Аудитория и ее активность

Месячная аудитория Instagram в России на 2021 год составляет 54 млн [(источник)](https://blog.hootsuite.com/instagram-demographics/)  
Дневная аудитория ~30 млн

На 2021 год в среднем, пользователь тратит 29 минут в день [(источник)](https://www.emarketer.com/content/emarketer-reduces-us-time-spent-estimates-for-facebook-and-snapchat)  
## 2.2. Собственные наблюдения
За 3 минуты просмотра ленты я успел просмотреть 30 постов, поставить 5 лайков и оставить один комментарий  

Будем предполагать, что средний пользователь добавляет 15 фотографий в свой профиль за месяц. Взаимодействует с подписками 10 раз в месяц  

## 2.3. Активность на пользователя

### Картинки
Картинки загружаются отдельными запросами на каждый пост. Причем на каждый пост приходится две картинки - аватар автора и сам пост.
```
60 GET / 3 min
```

### Post:
За один запрос на посты приходит 12 штук
```3 GET / 3 min``` Количество лайков, комментариев и сами первые три комментарии приходят вместе с запросом  
```15 POST / month``` Запрос на один пост занимает примерно 200кб (картинка) + 5 кб (другая информация)  

### Profile:
```1 GET / 3min``` Вместе с запросом на профиль приходит информация о подписках.  

Запрос на профиль занимает 40кб + первые посты ~740кб  

### Comment:
```
1 POST / 3 min.
5 GET / 3 min
```

Средняя длина комментариев в социальных сетях 200 символов (200b) [(источник)](https://habr.com/ru/post/72185/)  

### Subscriptions:
```
10 POST / month
```

### Likes:
```
5 POST / 3 min
```

## 2.4. Нагрузка от всей аудитории
Подитожим за сутки, учитывая, что пользователь тратит в сутки в среднем 29 мин:  

Пусть ```k = 30 000 000 users / (24 * 60 * 60) = 347,22 u/s``` - коэффициент, переводящий показатель requests per day для одного человека в rps для всей аудитории. 

### Получение картинок:
Средний вес картинки 200 Кб, аватарки - 4 Кб
```
60 * (29 min / 3 min) * k= 580 requests / day * k = 201 389 rps (19,59 Gb/s)
```

### Post
	GET: 3 * (29 min / 3 min) * k = 29 requests/day * k = 10 069 rps  (19,69 Gb/s)
	POST: 15 / 30 days * k = 0,5 requests/day * k = 174 rps (34,62 Mb/s)
### Profile
	GET: 1 * (29 min / 3 min) * k = 10 requests/day * k = 3 472 rps (32,78 Mb/s)

Прирост месячной аудитории в России за 2021 год согласно [источнику](https://blog.hootsuite.com/instagram-demographics/) составляет 3 млн пользователей.

Это значит, прирост пользователей в день ```3 million / 365 = 8219,14 users/day```  

Тогда rps на post profile будет ~ 1 rps  

### Comments:
За один запрос на комментарии приходит 12 комментариев. Один ответ весит в среднем 4 Kb (with gzip) -> 8kb (without gzip) -> avg comment response size = 8 / 12 = 0,67 Kb
```
GET 5 * 29 min / 3 min * k = 16 782 rps (65,56 Mb/s) (with gzip)
POST 1 * 29 min / 3 min * k = 3 356 rps (2,2 Mb/s) (without gzip)
```

### Likes:
```
POST 5 * 29 min / 3 min * k = 16 782 rps
```

### Subscriptions:
```
POST: 10 * 1 month / 30 days * k = 115 rps
```
Итого:
|Сущность|GET RPS|POST RPS|
|--------|-------|--------|
|Картинки|201 389|175|
|Post|10 069|174|
|Profile|3 472|1|
|Comment|16 782|3 356|
|Subscription|-|115|
|Like|-|16 782|
# 3. База данных
## 3.1. Логическая схема БД
![](./media/dbLogic.png)
## 3.2. Физическая схема БД
### 3.2.1. Фотографии
Все фотографии будем хранить в S3. В качестве S3 выберем решение от min.io.
В качестве S3 будем использовать решение minio  
Картинки постов будут храниться по пути /posts/{user_id}/{image_hash}.jpg
Аватарки - /avatars/{user_id}.jpg
#### Посты
При загрузке постов фотографии будут сжиматься до максимального размера ~700 Кб.  
Чаще всего встречающиеся размеры фотографий в Instagram: (720x1280, 1280x1350, 1080x1080, 720x720, 1080x695). Максимальный вес встретившейся фотографии - 623 кб. В среднем фотографии весят 200 кб  
Также будем хранить копию в размере 480x480 для отображения сетки профиля. Примерный вес такого изображения - 40 кб
#### Аватарки
Аватарки будут храниться в размере 150x150. В среднем вес аватраки - 5 кб  
### 3.2.2. Выбор СУБД
Данные пользователей, посты и комментарии будем хранить в PostgreSQL, как в наиболее надежной и функциональной реляционной БД.  
Данные сессии, лайки и подписки будем хранить в Redis из-за его выскокой скорости работы. Также Redis поддерживает неблокирующую master-slave репликацию.  
### 3.2.3. Шардирование и репликация
|Сущность|Признак шардирования|
|--------|--------------------|
|sessions |value|
|users    |id|
|posts|author_id|
|comments|post_id|
|likes|post_id|
|subscriptions|author_id|
|images|user_id|

Для обеспечения отказоустойчивости используем master-slave репликацию. По 2 реплики на каждый сервер. Мастер будет принимать запросы на запись, реплики - на чтение. При выходе из строя мастера, одна из реплик возмет на себя запросы на изменение. При выходе из строя всех реплик, запросы будут приходить только на мастер.
## 3.3. Размер данных
### Фотографии
Аватарки на всех пользователей
```
avatars = 54 000 000 * 5 kb = 257,49 Gb
avatars_speed = 1 rps * 24h * 60m * 60s = 0,5 Tb/month
posts_speed = 174 rps * 24h * 60m * 60s = 2,8 Tb/day = 84Tb/month
overall = 84,5 TB/month
```
### Сессии
Учитываем активную дневную аудиторию, т.к. у пользователей которые долго не заходят сессии пропадают
```
(4b (id) + 4b (user_id) + 64b (value) + 4b (expired_at)) * 30 000 000 = 2,12 Gb
```
### Пользователи
Учитываем месячную аудиторию
```
(4b (id) + 32b (username) + 60b (email) + 32b (name) + 256b (description) + 1b (gender) + ~120b (avatar url)) * 54 000 000 000 = 25,4 Gb

1rps * 24 * 60 * 60 = 41,61 Mb/day = 1,22 Gb/month
```
### Посты
```
(4b (id) + 4b (author_id) + ~120b (image url) + 512b (description) + 4 (likes_count) + 4 (comments_count) + 4 (created_at)) * 174 rps * 24h * 60m * 60s = 9,13 Gb/day = 273,86 Gb/month
```
### Комментарии
В среднем у постов 4 комментариев [(источник)](https://blog.cybermarketing.ru/vovlechennost-instagram/)
```
(4b (id) + 4b (author_id) + 4b (post_id) + 200b (text) * 4b (created_at)) * 4 * 174 rps * 24 * 60 * 60 = 12 Gb/day = 362,9 Gb/month
```
### Лайки
В среднем у поста 100 лайков [(источник)](https://blog.cybermarketing.ru/vovlechennost-instagram/)
```
(4b (id) + 4b (author_id) + 4b (post_id)) * 100 * 174 * 24 * 60 * 60 = 16 Gb/day = 504,04 Gb/month
```
### Подписки
```
(4b (id) + 4b (author_id) + 4b (following_id)) * 115 * 24 * 60 * 60 = 113,71 Mb/day = 3,33 Gb/month
```
|Сущность|Размер|
|---|---|
|Фотографии|0,25 Tb + 84,5 Tb/month|
|Сессии|2,12 Gb|
|Пользователи|25,4Gb + 1,22 Gb/month|
|Посты|273,86 Gb/month|
|Комментарии|362,9 Gb/month|
|Лайки|504,04 Gb/month|
|Подписки|3,33 Gb/month|
# n. Схема проекта
![](./media/projArch.jpeg)
