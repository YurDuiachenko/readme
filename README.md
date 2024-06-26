# Разговорный чат-бот для пассажиров РЖД

В последнее время, с появлением больших языковых разговорных моделей 
(Large Language Models, LLM), таких, как Yandex GPT, актуальным становится вопрос 
создания предметно-ориентированных чат-ботов, т.е. таких разговорных чат-ботов, которые 
способны поддерживать беседу в рамках какой-то узкой предметной области. В итоге мы решили 
сконцентрироваться на железнодорожной отрасли, для которой и будем разрабатывать такого чат-бота.

## Технологии
В основе [чат-бота](https://github.com/yandex-datasphere/yatalks-potter-bot) использовалась технология [RAG(Retrieval-Augmented Generation)](https://github.com/yandex-datasphere/yatalks-potter-bot) при комбинации библиотек [LangChain](https://www.langchain.com/) и [YandexChain](https://github.com/yandex-datasphere/yandex-chain), для эмбедингов использовались YandexEmbedings, в качестве векторного хранилища был выбран [LanceDB](https://lancedb.com/), в качестве основного LLM - **YandexGPT** (хотя и были неудачные попытки построить работу на HuggingFace). 
[API Яндекс.Расписаний](https://yandex.ru/dev/rasp/raspapi/) было использовано для получения актуальной инофрмации о поездах и маршрутах.
**Telebot API** помог в написании обёртки чат-бота в телеграм бота.

## Этапы работы

* [Сбор и подготовка данных](#сбор-и-подготовка-данных) 
* [Подготовка проекта](#подготовка-проекта)
* [Retrieval-Augmented Generation (RAG)](#retrieval-augmented-generation)
* [Yandex API Расписаний](#yandex-api-расписаний)
* [Телеграм-бот](#телеграм-бот)
* [Тестирование и оценка](#тестирование-и-оценка)

### Сбор и подготовка данных
Процесс сбора данных с **[веб-сайта](https://www.rzd.ru/)** включает в себя ряд технических этапов, 
начиная с извлечения информации о расписаниях поездов, статусах отправлений, билетных ценах и заканчивая другими связанными с РЖД данными. 

Полученные данные направляются на этап обработки с использованием специализированных инструментов, таких как **[Gemini](https://gemini.google.com/?hl=ru)** или **[Claude](https://www.anthropic.com/claude)**. 

Эти инструменты предназначены для автоматического анализа текста и его классификации на более мелкие единицы (т.е. данные инструменты помогают 
уменьшенить объема текста с без потери смысла, а именно: удалить из текста излишние слова, фразы или абзацы, сохраняя 
при этом основную информацию и ключевые концепции). Такой подход позволяет улучшить качество данных и сделать их более компактными 
для последующей работы.

Подготовленные данные сохраняются в текстовые файлы формата 
TXT, который является универсальным форматом хранения текстовой информации.

### Подготовка проекта 

#### Установка библиотек 

```cmd
pip install langchain==0.1.0 sentence_transformers==2.5.1 lancedb==0.6.0 unstructured==0.12.5 yandex_chain==0.0.7 yandexcloud==0.262.0
pip install telebot requests
```

#### Импорт модулей

```
import pandas as pd
import numpy as np
import seaborn as sns
import matplotlib.pyplot as plt
import langchain
import langchain.document_loaders
import langchain.text_splitter as sp
from yandex_chain import YandexEmbeddings
import lancedb
import telebot
import datetime
import json
import requests
```

#### Получение ключей из окружения

```
api_key = os.environ['apy']
rasp_api_key = os.environ['rasp_api_key']
telebot_api_key = os.environ['telebot_api_key']
```
TODO: описать для чего ключи

### Retrieval-Augmented Generation
TODO: описать рэг
![image](https://github.com/YurDuiachenko/readme/assets/72216941/40121c3f-2a0e-40be-9f84-77631fc72920)

![image](https://github.com/YurDuiachenko/readme/assets/72216941/0a0ba9ff-aea6-4177-bfc4-4e8ed91ca966)


### Yandex API Расписаний

Теперь перейдем к работе с *Yandex API Расписаний*. 

Теперь создадим переменную, хранящую сообщение об ошибке, возникающей при обработке данных. 
```jupyter
error_message = 'Ошибка при обработке данных. Попробуйте сделать запрос еще раз!'
```

*Необходимые функции:* 
1. ```make_request``` Эта функция отправляет HTTP запрос к API Yandex Расписаний и возвращает результат в формате JSON.
2. ```get_city_code``` Эта функция принимает на вход название населенного пункта и возвращает его код в соответствии с API Yandex Расписаний.
3.  ```get_routes``` Эта функция ищет маршруты от одного населенного пункта к другому. Она использует API Yandex Расписаний для поиска маршрутов, фильтрует результаты и возвращает информацию о маршрутах.
4. ```get_routes_by_date``` Эта функция аналогична get_routes, но позволяет искать маршруты по указанной дате.

> Реализацию вышеуказанных функций можно посмотреть в файле (ссылка на файл в git)


### Телеграм-бот

Создадим *экземпляр* бота с использованием загруженного *ключа API*.
```jupyter
bot = telebot.TeleBot(telebot_api_key)
```

Определяем *кнопки* клавиатуры, которые будут использоваться для взаимодействия с пользователем.
```jupyter
api_button = types.KeyboardButton('📓 Режим API')
ai_button = types.KeyboardButton('🤖 Режим AI')
home_button = types.KeyboardButton('🏠 Вернуться на главную')
back_button = types.KeyboardButton('🔙 Вернуться назад')
delete_button = types.KeyboardButton('🗑 Очистить чат')
```

Далее прописываем *обработчики* для кнопок и обработчик для команды ```/start```.
Можно посмотреть в файле (ссылка на файл в git)

Следущим шагом запускаем бота.

### Тестирование и оценка
TODO: описать тестирование
## Команда проекта

* [Брежнева Алена](https://github.com/alenka192003)
* [Васильев Владимир](https://github.com/SilentMiver) 
* [Дьяченко Юрий](https://github.com/YurDuiachenko)
* [Замуруев Роман](https://github.com/Zamuruev)
* [Карпушин Андрей](https://github.com/recwayer)
* [Левшенко Денис](https://github.com/kottzi)

## Источники
TODO добавить источники
* [Поттер-бот](https://github.com/yandex-datasphere/yatalks-potter-bot)
* [Retrieval-Augmented Generation](https://habr.com/ru/articles/772130/)
* [Yandex API Расписаний](https://yandex.ru/dev/rasp/raspapi/)
