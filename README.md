# Проектирование высоконагруженного сайта объявлений

Курсовая работа в рамках 3-го семестра программы по Веб-разработке ОЦ VK x МГТУ им. Н.Э. Баумана (ex. "Технопарк") по дисциплине "Проектирование высоконагруженных сервисов"

#### Автор - [Максим Сиканов](https://park.vk.company/profile/m.sikanov/ "Страница на портале VK x МГТУ")
#### Задание - [Методические указания](https://github.com/init/highload/blob/main/homework_architecture.md)

#### Содержание:
1. [Тема, функционал и аудитория](#1)
2. [Расчёт нагрузки](#2)
3. [Глобальная балансировка нагрузки](#3)
4. [Локальная балансировка нагрузки](#4)
5. [Логическая схема базы данных](#5)
6. [Физическая схема базы данных](#6)
7. [Алгоритмы](#7)
8. [Технологии](#8)
9. [Обеспечение надёжности](#9)
10. [Схема проекта](#10)
11. [Расчёт ресурсов](#11)


## Часть 1. Тема и целевая аудитория <a name="1"></a>

### Тема курсовой работы - **"Проектирование сайта объявлений"**
В качестве примера и аналога выбран ведущий в России сайт объявлений - [Avito](https://www.avito.ru/)

### Ключевой функционал сервиса
- Регистрация и авторизация пользователей
- Создание / редактирование / поиск / просмотр объявлений
- Возможность оставлять отзывы

### Ключевые продуктовые решения
- Поиск (по фильтрам)

### Целевая аудитория
- 61 млн активных пользователей в месяц в странах СНГ [^1]
- В среднем пользователи проводят на Авито 11 минут в месяц [^1]
- 97% трафика поступает на Авито из России [^1]
- Демография: 56.79% мужчин и 43.21% женщин [^1]

![img.png](src/age.png)

## Часть 2. Расчёт нагрузки <a name="2"></a>

### Продуктовые метрики

MAU - 61 млн пользователей [^1]

DAU - 2 млн пользователей [^1]

#### Среднее количество действий пользователя по типам в день:
- В Авито создается ≈ `1.4 млн` объявлений в день [^1] => `0.0226 объявлений/сут` на человека
- В Авито происходит ≈ `8` сделок в секунду [^1] => `0.0113 сделок/сут` на человека
- посещаемость Авито более `384 млн` пользователей в месяц [^1] => `0.21 посещений/сут` на человека

### Технические метрики

#### Средний размер харанилища пользователя по типам:
| Хранимые данные                                 | Оценочный размер на пользователя |
|-------------------------------------------------|----------------------------------|
| Персональные данные (ФИО, почта, пароль и т.д.) | `1 КБ`                           |
| Аватар                                          | `256 КБ`                         |
| Объявление                                      | `1 МБ`                           |
| Отзыв                                           | `1 КБ`                           |

- Объявление в среднем содержит 3-4 фото (размер фото в среднем `256 КБ`) и текстовое описание. Итого ≈ `1 МБ`
- В среднем на одного пользователя приходится 5 объявления и 10 отзывов [^1]

Возьмём общее оценочное число пользователей с запасом = `80 млн` - будем использовать далее в расчётах.

Тогда общий размер хранилища в худшем случае: `80 млн пользователей * (5 МБ на объявления + 256 КБ на аватар + 1 КБ персональные данные + 10 КБ отзывы) ≈ 249 ТБ`

За год можно ожидать прирост пользователей до `13 %` => `249 ТБ * 0.13 = 32 ТБ 40 ГБ` нового пространстра может потребоваться.

#### RPS и сетевой трафик по типам запросов:
| Тип запроса               | Средний оценочный RPS | Пиковое потребление, Гбит/с | Суммарный суточный трафик, Тб/сутки |
|---------------------------|-----------------------|-----------------------------|-------------------------------------|
| Создание объявления       | `17`                  | `0.03`                      | `1.4`                               |
| Редактирование объявления | `139`                 | `0.24`                      | `11.45`                             |
| Поиск объявлений          | `140`                 | `0.0001`                    | `0.011`                             |
| Просмотр объявлений       | `14000`               | `27.34`                     | `1153.6`                            |
| Создание отзыва           | `5`                   | `-`                         | `0.0004`                            |
| Просмотр отзывов          | `420`                 | `0.0008`                    | `0.034`                             |

**Расчёты RPS:**
- Создание объявления: `1.4 млн объявлений в сутки / (24 * 3600) ~= 17 RPS`
- Редактирование объявления: `12 млн объявлений в сутки / (24 * 3600) ~= 139 RPS` [^2]
- Поиск объявлений: `2 млн DAU * 6 / (24 * 3600) ~= 140 RPS` при условии, что каждый сделает 6 запросов
- Просмотр объявлений: `2 млн DAU * 6 * 100 / (24 * 3600)  ~= 14000 RPS` при условии, что каждый поиск будет сожержать 100 объявлений
- Создание отзыва: `10 сделок/с * 0.5 = 5 RPS` при условии, что каждый второй будет оставлять отзыв
- Просмотр отзывов: `140 поиск/с * 3 = 420 RPS` при условии, за один поиск человек откроет отзывы 3 раза

**Расчёты трафика:**
- Средний: API + Статика: `RPS * средний размер в ГБ = X Гбит/с`
- Пиковый (Пиковый коэф трафика от среднего с запасом = 2): API + Статика: `2 * X Гбит/с`
- Суммарный суточный: API + Статика: `(X / 1024) * (24 * 3600) с/сут = Y Тб/сут`

|          | Суммарный RPS | Суммарное пиковое потребление, Гбит/с | Суммарный суточный трафик, Тб/сутки |
|----------|---------------|---------------------------------------|-------------------------------------|
| **Итог** | `14721`       | `27.61`                               | `1166.5`                            |


## Часть 3. Глобальная балансировка нагрузки <a name="3"></a>

Для обеспечения минимального latency основную часть дата-центров следует размещать на территории, наиболее близкой к наибольшему количеству пользователей.
Так как в случае Авито 97% заказов приходится на российский рынок[^1], то ЦОДы размещать стоит в наиболее густонаселённых регионах РФ.

![population.png](./src/population.png)

Поэтому были выбраны следующие города для установки датацентров:
* Москва
* Санкт-Петербург
* Екатеринубрг
* Иркутск

![image](https://github.com/Max425/tp-highload-avito/assets/57855981/7666c272-537c-4e49-ba93-e2264f10d568)


### Методы глобальной балансировки

Для глобальной балансировки запросов и нагрузки будем использовать:

С помощью GeoDNS будем определять физически ближайший к пользователю ЦОД. ЦОД-ы будут отвечать за следующие районы:
* Москва -> Москва, Московская область
* Санкт-Петербург -> Санкт-Петербург, Ленинградская область
* Екатеринбург -> Средняя часть России
* Иркутск -> Сибирь, Восток

Далее будем балансировать запросы между ЦОДами с помощью Routing - BGP Anycast

## Часть 4. Локальная балансировка нагрузки <a name="4"></a>


## Часть 5. Логическая схема базы данных <a name="5"></a>


## Часть 6. Физическая схема базы данных <a name="6"></a>


## Часть 7. Алгоримты <a name="7"></a>


## Часть 8. Технологии <a name="8"></a>


## Часть 9. Обеспечение надёжности <a name="9"></a>


## Часть 10. Схема проекта <a name="10"></a>


## Часть 11. Расчёт ресурсов <a name="11"></a>


### Список источников:
[^1]: [Статистика Авито в 2024 году](https://inclient.ru/avito-stats/#avito)

[^2]: [Авито Playbook](https://github.com/avito-tech/playbook)
