# Домашнее задание к занятию 5. «Практическое применение Docker»

### Инструкция к выполнению

1. Для выполнения заданий обязательно ознакомьтесь с [инструкцией](https://github.com/netology-code/devops-materials/blob/master/cloudwork.MD) по экономии облачных ресурсов. Это нужно, чтобы не расходовать средства, полученные в результате использования промокода.
3. **Своё решение к задачам оформите в вашем GitHub репозитории.**
4. В личном кабинете отправьте на проверку ссылку на .md-файл в вашем репозитории.
5. Сопроводите ответ необходимыми скриншотами.

---
## Примечание: Ознакомьтесь со схемой виртуального стенда [по ссылке](https://github.com/netology-code/shvirtd-example-python/blob/main/schema.pdf)

---

## Задача 0
1. Убедитесь что у вас НЕ(!) установлен ```docker-compose```, для этого получите следующую ошибку от команды ```docker-compose --version```
```
Command 'docker-compose' not found, but can be installed with:

sudo snap install docker          # version 24.0.5, or
sudo apt  install docker-compose  # version 1.25.0-1

See 'snap info docker' for additional versions.
```
В случае наличия установленного в системе ```docker-compose``` - удалите его.  
2. Убедитесь что у вас УСТАНОВЛЕН ```docker compose```(без тире) версии не менее v2.24.X, для это выполните команду ```docker compose version```  
###  **Своё решение к задачам оформите в вашем GitHub репозитории!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!**

## Решение Задача 0

```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce docker-ce-cli containerd.io
sudo systemctl start docker
sudo systemctl enable docker
docker compose version
```
![image](https://github.com/user-attachments/assets/e90ba025-790e-4c78-ba0b-43dda053e0d6)

---

## Задача 1
1. Сделайте в своем github пространстве fork [репозитория](https://github.com/netology-code/shvirtd-example-python/blob/main/README.md).
   Примечание: В связи с доработкой кода python приложения. Если вы уверены что задание выполнено вами верно, а код python приложения работает с ошибкой то используйте вместо main.py файл old_main.py(просто измените CMD)
3. Создайте файл с именем ```Dockerfile.python``` для сборки данного проекта(для 3 задания изучите https://docs.docker.com/compose/compose-file/build/ ). Используйте базовый образ ```python:3.9-slim```. 
Обязательно используйте конструкцию ```COPY . .``` в Dockerfile. Не забудьте исключить ненужные в имадже файлы с помощью dockerignore. Протестируйте корректность сборки.  
4. (Необязательная часть, *) Изучите инструкцию в проекте и запустите web-приложение без использования docker в venv. (Mysql БД можно запустить в docker run).
5. (Необязательная часть, *) По образцу предоставленного python кода внесите в него исправление для управления названием используемой таблицы через ENV переменную.
---
### ВНИМАНИЕ!
!!! В процессе последующего выполнения ДЗ НЕ изменяйте содержимое файлов в fork-репозитории! Ваша задача ДОБАВИТЬ 5 файлов: ```Dockerfile.python```, ```compose.yaml```, ```.gitignore```, ```.dockerignore```,```bash-скрипт```. Если вам понадобилось внести иные изменения в проект - вы что-то делаете неверно!

## Решение Задача 1

```bash
mkdir 05-virt-04-docker-in-practice-hw
cd 05-virt-04-docker-in-practice-hw
git clone https://github.com/netology-code/shvirtd-example-python.git
```

Создаем файл Dockerfile.python

```bash
cat > Dockerfile.python <<EOF
# Используем базовый образ Python 3.9 slim
FROM python:3.9-slim

# Устанавливаем рабочую директорию
WORKDIR /app

# Копируем все файлы в рабочую директорию
COPY . .

# Устанавливаем зависимости
RUN pip install --no-cache-dir -r requirements.txt

# Указываем команду для запуска приложения
CMD ["python", "main.py"]
EOF
```
Создаем файл .dockerignore

```bash
cat > .dockerignore <<EOF
__pycache__
*.pyc
*.pyo
.git
.gitignore
EOF
```

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
docker run --name mysql-container -e MYSQL_ROOT_PASSWORD="YtReWq4321" -e MYSQL_DATABASE="virtd" -e MYSQL_USER="app" -e MYSQL_PASSWORD="QwErTy1234" -p 3306:3306 -d mysql:latest
docker ps -a
python main.py
```

![image](https://github.com/user-attachments/assets/96e0ae92-9d21-48b4-8c8b-694be68040ec)

Модифицируем файл main.py

```python
from flask import Flask
from flask import request
import os
import mysql.connector
from datetime import datetime

app = Flask(__name__)
db_host=os.environ.get('DB_HOST')
db_user=os.environ.get('DB_USER')
db_password=os.environ.get('DB_PASSWORD')
db_database=os.environ.get('DB_NAME')
table_name = os.environ.get('TABLE_NAME', 'default_table_name')


# Подключение к базе данных MySQL
db = mysql.connector.connect(
host=db_host,
user=db_user,
password=db_password,
database=db_database,
autocommit=True )
cursor = db.cursor()

# SQL-запрос для создания таблицы в БД
create_table_query = f"""
CREATE TABLE IF NOT EXISTS {db_database}.{table_name} (
id INT AUTO_INCREMENT PRIMARY KEY,
request_date DATETIME,
request_ip VARCHAR(255)
)
"""
cursor.execute(create_table_query)

@app.route('/')
def index():
    # Получение IP-адреса пользователя
    ip_address = request.headers.get('X-Forwarded-For')

    # Запись в базу данных
    now = datetime.now()
    current_time = now.strftime("%Y-%m-%d %H:%M:%S")
    query = "INSERT INTO requests (request_date, request_ip) VALUES (%s, %s)"
    values = (current_time, ip_address)
    cursor.execute(query, values)
    db.commit()

    return f'TIME: {current_time}, IP: {ip_address}'


if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0')
```
Изменяем название таблицы

```bash
export TABLE_NAME='new_requests'
```

Проверка

![image](https://github.com/user-attachments/assets/6aa27fc7-9977-4fd2-91e3-626be7a5f38c)

---

## Задача 2 (*)
1. Создайте в yandex cloud container registry с именем "test" с помощью "yc tool" . [Инструкция](https://cloud.yandex.ru/ru/docs/container-registry/quickstart/?from=int-console-help)
2. Настройте аутентификацию вашего локального docker в yandex container registry.
3. Соберите и залейте в него образ с python приложением из задания №1.
4. Просканируйте образ на уязвимости.
5. В качестве ответа приложите отчет сканирования.

## Решение Задача 2 (*)

Авторизация в Yandex Cloud (token https://oauth.yandex.ru/authorize?response_type=token&client_id=1a6990aa636648e9b2ef855fa7bec2fb)

```bash
yc init
```
![image](https://github.com/user-attachments/assets/766c8120-0fa0-481e-b977-8d3e0913f3e7)


```bash
yc container registry create --name test
```
![image](https://github.com/user-attachments/assets/52234e85-ceaf-41da-adc7-e829699a526c)

```bash
yc container registry configure-docker
```
![image](https://github.com/user-attachments/assets/63ddda1f-8da7-45b1-94ba-a1a783051173)

```bash
docker build -t cr.yandex/crphlbn0nvu5j1v4u5qo/python-app:latest .
```
![image](https://github.com/user-attachments/assets/597de0d5-4ca2-4f82-9a71-0c4deaf16693)

```bash
docker push cr.yandex/crphlbn0nvu5j1v4u5qo/python-app:latest
```
![image](https://github.com/user-attachments/assets/b1a255c9-3be9-4dea-9c5a-18e623792e4e)


При сканировании возникла проблема 

Проверяем образы
```
yc container image list
```
![image](https://github.com/user-attachments/assets/2f1a5d04-a4b0-4b8b-af13-d790633972eb)

![image](https://github.com/user-attachments/assets/9773c7b0-9ffa-4c5c-abff-e25c4034bdd2)

Сканируем

```bash
yc container image scan cr.yandex/crphlbn0nvu5j1v4u5qo/python-app:latest
```
Ошибка, не найден образ, но образ точно есть, несколько раз перезаливал его в реестр.

![image](https://github.com/user-attachments/assets/3eea8447-99c0-4993-b3e3-b4999975a93b)

Отсканировал через Web-интефейс

![image](https://github.com/user-attachments/assets/78027827-ee7f-44b2-9403-cebd25436df8)


## Задача 3
1. Изучите файл "proxy.yaml"
2. Создайте в репозитории с проектом файл ```compose.yaml```. С помощью директивы "include" подключите к нему файл "proxy.yaml".
3. Опишите в файле ```compose.yaml``` следующие сервисы: 

- ```web```. Образ приложения должен ИЛИ собираться при запуске compose из файла ```Dockerfile.python``` ИЛИ скачиваться из yandex cloud container registry(из задание №2 со *). Контейнер должен работать в bridge-сети с названием ```backend``` и иметь фиксированный ipv4-адрес ```172.20.0.5```. Сервис должен всегда перезапускаться в случае ошибок.
Передайте необходимые ENV-переменные для подключения к Mysql базе данных по сетевому имени сервиса ```web``` 

- ```db```. image=mysql:8. Контейнер должен работать в bridge-сети с названием ```backend``` и иметь фиксированный ipv4-адрес ```172.20.0.10```. Явно перезапуск сервиса в случае ошибок. Передайте необходимые ENV-переменные для создания: пароля root пользователя, создания базы данных, пользователя и пароля для web-приложения.Обязательно используйте уже существующий .env file для назначения секретных ENV-переменных!

2. Запустите проект локально с помощью docker compose , добейтесь его стабильной работы: команда ```curl -L http://127.0.0.1:8090``` должна возвращать в качестве ответа время и локальный IP-адрес. Если сервисы не стартуют воспользуйтесь командами: ```docker ps -a ``` и ```docker logs <container_name>``` . Если вместо IP-адреса вы получаете ```NULL``` --убедитесь, что вы шлете запрос на порт ```8090```, а не 5000.

5. Подключитесь к БД mysql с помощью команды ```docker exec <имя_контейнера> mysql -uroot -p<пароль root-пользователя>```(обратите внимание что между ключем -u и логином root нет пробела. это важно!!! тоже самое с паролем) . Введите последовательно команды (не забываем в конце символ ; ): ```show databases; use <имя вашей базы данных(по-умолчанию example)>; show tables; SELECT * from requests LIMIT 10;```.

6. Остановите проект. В качестве ответа приложите скриншот sql-запроса.

## Решение Задача 3

Файл compose.yaml

```yaml
version: '3.8'

# Включаем файл proxy.yaml
include:
  - proxy.yaml

services:
  web:
    build:
      context: .
      dockerfile: Dockerfile.python
    # image: cr.yandex/crphlbn0nvu5j1v4u5qo/python-app:latest
    restart: always
    networks:
      backend:
        ipv4_address: 172.20.0.5
    environment:
      - DB_HOST=db
      - DB_USER=${DB_USER}
      - DB_PASSWORD=${DB_PASSWORD}
      - DB_NAME=${DB_NAME}

  db:
    image: mysql:8
    restart: always
    networks:
      backend:
        ipv4_address: 172.20.0.10
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=${DB_NAME}
      - MYSQL_USER=${DB_USER}
      - MYSQL_PASSWORD=${DB_PASSWORD}

networks:
  backend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24
```

```bash
docker-compose up -d
docker ps -a
```
![image](https://github.com/user-attachments/assets/acb33e02-e27c-44b9-a399-87a8ff7e7978)


```bash
curl -L http://127.0.0.1:8090; echo
```
![image](https://github.com/user-attachments/assets/c114555c-c07d-4ad1-94d1-bc81df8e3227)

```bash
docker exec -it b269cc57ffd2 mysql -u app -p
```

```sql
SHOW DATABASES; USE virtd; SHOW TABLES; SELECT * FROM requests LIMIT 10;
```

![image](https://github.com/user-attachments/assets/63949ae9-2862-42c0-81e0-027028c1dc19)

```bash
docker compose down
```
![image](https://github.com/user-attachments/assets/9323f1f1-919b-486a-b323-154a03a22ac2)


## Задача 4
1. Запустите в Yandex Cloud ВМ (вам хватит 2 Гб Ram).
2. Подключитесь к Вм по ssh и установите docker.
3. Напишите bash-скрипт, который скачает ваш fork-репозиторий в каталог /opt и запустит проект целиком.
4. Зайдите на сайт проверки http подключений, например(или аналогичный): ```https://check-host.net/check-http``` и запустите проверку вашего сервиса ```http://<внешний_IP-адрес_вашей_ВМ>:8090```. Таким образом трафик будет направлен в ingress-proxy. ПРИМЕЧАНИЕ:  приложение(old_main.py) весьма вероятно упадет под нагрузкой, но успеет обработать часть запросов - этого достаточно. Обновленная версия (main.py) не прошла достаточного тестирования временем, но должна справиться с нагрузкой.
5. (Необязательная часть) Дополнительно настройте remote ssh context к вашему серверу. Отобразите список контекстов и результат удаленного выполнения ```docker ps -a```
6. В качестве ответа повторите  sql-запрос и приложите скриншот с данного сервера, bash-скрипт и ссылку на fork-репозиторий.

## Решение Задача 4

ssh tenda@51.250.102.36

```bash
sudo yum install -y yum-utils git
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce docker-ce-cli containerd.io
sudo systemctl start docker
sudo systemctl enable docker
```

bash-скрипт deploy.sh

```bash
#!/bin/bash

# Проверка, что скрипт выполняется с правами суперпользователя
if [ "$EUID" -ne 0 ]; then
  echo "Пожалуйста, запустите скрипт с правами суперпользователя (sudo)."
  exit 1
fi


# Переменные
REPO_URL="https://github.com/killakazzak/shvirtd-example-python.git"
TARGET_DIR="/opt/my_project/"

# Скачивание репозитория
if [ ! -d "$TARGET_DIR" ]; then
  echo "Клонирование репозитория..."
  git clone "$REPO_URL" "$TARGET_DIR"
else
  echo "Репозиторий уже существует в $TARGET_DIR. Обновление..."
  cd "$TARGET_DIR" || exit
  git pull origin main
fi

# Переход в каталог проекта
cd "$TARGET_DIR" || exit

# Запуск Docker Compose
echo "Запуск проекта с помощью Docker Compose..."
docker compose up -d

echo "Проект запущен."
```
```bash
chmod +x deploy.sh
./deploy.sh
docker ps -a
```
![image](https://github.com/user-attachments/assets/af6d5660-b64d-4676-b695-f7c517304ad1)

![image](https://github.com/user-attachments/assets/0a76b256-48b9-4bde-be53-95b095db0c14)

![image](https://github.com/user-attachments/assets/bf6e7a42-c9bb-413f-a80b-b17fb3f17b92)


На удаленном сервере

```bash
sudo usermod -aG docker $USER
```
На локальном сервере

```bash
docker context create my-remote --docker  "host=ssh://tenda@51.250.102.36"
docker context ls
docker context use my-remote
docker ps
```
![image](https://github.com/user-attachments/assets/265f197c-808e-47d2-a3d8-24c64f9c8920)

![image](https://github.com/user-attachments/assets/c8101420-8465-4804-b30e-2f46fbf09f69)

```bash
docker exec -it 370fc4e9084e mysql -u app -p
```
```sql
SHOW DATABASES; USE virtd; SHOW TABLES; SELECT * FROM requests LIMIT 10;
```

![image](https://github.com/user-attachments/assets/c72629a8-712c-4f06-b509-f0948f1b261b)


https://github.com/killakazzak/shvirtd-example-python/tree/main


## Задача 5 (*)
1. Напишите и задеплойте на вашу облачную ВМ bash скрипт, который произведет резервное копирование БД mysql в директорию "/opt/backup" с помощью запуска в сети "backend" контейнера из образа ```schnitzler/mysqldump``` при помощи ```docker run ...``` команды. Подсказка: "документация образа."
2. Протестируйте ручной запуск
3. Настройте выполнение скрипта раз в 1 минуту через cron, crontab или systemctl timer. Придумайте способ не светить логин/пароль в git!!
4. Предоставьте скрипт, cron-task и скриншот с несколькими резервными копиями в "/opt/backup"

## Решение Задача 5 (*)

Создаем файл окружения .env (скрываем логин/пароль)

```bash
mkdir /opt/backup
cat <<EOL > /opt/backup/.env
DB_USER="app"
DB_PASSWORD="QwErTy1234"
DB_NAME="virtd"
EOL
```

Создаем скрипт backup

```bash
cat <<EOL > /root/backup.sh
#!/bin/bash

# Загрузка переменных окружения
source /opt/backup/.env

DB_HOST="db"
BACKUP_DIR="/opt/backup"
TIMESTAMP=$(date +"%Y%m%d%H%M")

# Создание резервной копии
docker run --rm \
  --network my_project_backend \
  -e MYSQL_ROOT_PASSWORD="$DB_PASSWORD" \
  schnitzler/mysqldump \
  -h "$DB_HOST" -u "$DB_USER" -p"$DB_PASSWORD" "$DB_NAME" > "$BACKUP_DIR/backup_$TIMESTAMP.sql"

# Удаление резервных копий старше 7 дней
find "$BACKUP_DIR" -type f -name "*.sql" -mtime +7 -exec rm {} \;
EOL
```
Запускаем скрип backup

```bash
chmod +x backup.sh
./backup.sh
```
```
crontab -e
* * * * * /opt/backup/backup.sh >> /opt/backup/backup.log 2>&1
```

![image](https://github.com/user-attachments/assets/0371db98-bc5d-44ee-b5de-f3cb746edbbc)


## Задача 6
Скачайте docker образ ```hashicorp/terraform:latest``` и скопируйте бинарный файл ```/bin/terraform``` на свою локальную машину, используя dive и docker save.
Предоставьте скриншоты  действий .

## Решение Задача 6

```bash
docker pull hashicorp/terraform:latest
docker save -o terraform_latest.tar hashicorp/terraform:latest
```
![image](https://github.com/user-attachments/assets/89b38a0f-41ba-45f5-89e7-b275971a05b4)



## Задача 6.1
Добейтесь аналогичного результата, используя docker cp.  
Предоставьте скриншоты  действий .

## Решение Задача 6.1

```bash
docker pull hashicorp/terraform:latest
docker run -d --name terraform_container hashicorp/terraform:latest tail -f /dev/null
```
![image](https://github.com/user-attachments/assets/73f0ae4e-951b-4927-8a5a-da92d25e9eed)

```bash
docker cp terraform_container:/bin/terraform ./terraform
```
![image](https://github.com/user-attachments/assets/56b546c4-09b3-4190-bf6c-b8047d09f04b)

```bash
docker rm -f terraform_container
```
![image](https://github.com/user-attachments/assets/8be3b368-5430-4847-9315-fb7ab0e99d05)

![image](https://github.com/user-attachments/assets/a2385b03-d0ea-492a-820d-4143c79c6195)

## Задача 6.2 (**)
Предложите способ извлечь файл из контейнера, используя только команду docker build и любой Dockerfile.  
Предоставьте скриншоты  действий .

## Решение Задача 6.2 (**)


Создаем временный контейнер

```bash
docker create --name temp_container hashicorp/terraform:latest
```
Копируем файл из временного контейнера на локальную машину:

```bash
docker cp temp_container:/bin/terraform ./terraform
```
Удаляем временный контейнер

```bash
docker rm temp_container
```
Затем создайте Dockerfile

```bash
cat > Dockerfile <<EOF
FROM scratch
COPY terraform /terraform
CMD ["/terraform"]
EOF
```
Собираем образ

```bash
docker build -t my_terraform_image .
```
Запускаем образ
```bash
docker run --name my_terraform_container my_terraform_image
```
Копируем файл из нового образа на локальную машину

```bash
docker cp my_terraform_container:/terraform ./terraform
```
Удаляем контейнер

```bash
docker rm my_terraform_container
```

## Задача 7 (***)
Запустите ваше python-приложение с помощью runC, не используя docker или containerd.  
Предоставьте скриншоты  действий .

## Решение Задача 7 (***)


