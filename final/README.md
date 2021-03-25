Введение
========
В этом файле описывается, как запустить тесты, где что находится, как включить, зачем использовалось и т.д. 

Структура проекта:
=================
- **.env_template** файл - шаблон для **.env** файла
- **docker-compose.yml** - файл для запуска основных контейнеров
- **jenkins.docker-compose.yml** - файл для запуска jenkins
- **tests.docker-compose.yml** - файл для запуска тестов в контейнере
- Папка **builds/** содержит папки с программами, из которых можно сдлеать образы для докера
  - **jenkins/** - тут Dockerfile для образа Jenkins с docker CLI, docker-compose, python3.8 и необходимыми для тестов библиотеками
  - **proxy/** - тут Dockerfile и конфиг для образа NGINX сервера, который будет проксировать запросы к приложению.
    Приложение не принимает API запросы с IP, отличного от IP пользователя (администратора). Ну, у меня выдавало 401 Not Authorized
  - **mock/**
  - **tests/** - тесты можно запустить как в контейнере, так и вне него
- Папка **scripts/** - bash-скрипты
- Папка **configs/** - тут всё, что остальное

Обычный запуск (без Jenkins)
============================
Подготовка к первому запуску
----------------------------
```shell script
# Создать и отредактировать файл .env
cp .env_template .env
vim .env

# Создать структуру папок mount и перенести конфиги
scripts/install.sh
``` 

Запуск
------
```shell script
# Поднять основные контейнеры
docker-compose up

# Так можно проверить, что контейнеры запущены
while test $(docker-compose ps | grep -E 'selenoid.*Up' | wc -l) -eq 0; do sleep 5; done
sleep 5

# Запустить тесты..

# .. или вне контейнера
scripts/run.sh

# .. или внутри контейнера
docker-compose -f tests.docker-compose.yml up --abort-on-container-exit

# Посмотреть результаты

allure serve mount/allure-results
```

Очистка
-------
Для последующих запусков тестов не нужно перезапускать основные контейнеры,
либо удалять содержимое базы данных в папке **mount/**. Тесты очистят таблицу сами

Запуск на Jenkins
=================
Поднять Jenkins
---------------
```shell script
docker-compose -f jenkins.docker-compose.yml up --build

# создание пайплайна
jenkins-jobs --conf configs/jenkins.ini update configs/pipeline.yaml

```

Настройка Jenkins
-----------------
Я не смог установить allure commandline установиться из Maven Central 2.13, но получилось установить из Maven Central 2.7

Пометка
-------
Я установил опцию `-m enable_video` в configs/pipeline.groovy, потому что тестов много, они примерно 20 минут у меня работают

Вопрос-ответ
============
Зачем нужен прокси?
------------------
Методом тыка обнаружено, что приложение требует сессионную куку 'session' для того, 
чтобы использовать API. И эта сессионная кука передаётся пользователю, который должен быть
активным. Также приложение требует совпадения User-Agent у API клиента и у UI клиента

После авторизации вручную и передачи User-Agent и session в requests.post() можно вызвать API функцию

Но если поместить тесты в контейнер, запустить браузер через селеноид
и получить от него session и User-Agent, вызвать API не получалось, 401 Not Authorized

Значит, приложение требует совпадения IP у API клиента и UI клиента

Через proxy всё заработало как часы (заодно разобрался, как устроен конфиг NGINX)

Про опции у тестов (возможные аргументы pytest)
-----------------------------------------------------
Только две кастомные опции: `--time-load-page` и `--time-input-text`.
Они задают максимальное время загрузки страницы и ввода текста.
Наверняка их не понадобится менять

Ещё можно включить/отключить опцию записи видео через переменную `ENABLE_VIDEO` в **.env** файле 

Как посмотреть видео в allure отчёте?
------------------------------------
Нужно установить переменной `ENABLE_VIDEO` в **.env** файле значение `+`.
Это обрабатывается в файле builds/tests/code/settings.py

Чтобы для UI теста записать видео, надо пометить тест маркером **enable_video**

В следующем примере в allure репорт к тесту прикрепится видео-фрагмент, в котором открывается страница регистрации
и выполняется переход на страницу авторизации
```python
import pytest

@pytest.mark.enable_video
def test_example(registration_page):
    registration_page.go_to_authorization_page()
```