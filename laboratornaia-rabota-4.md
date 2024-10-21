# Лабораторная работа 4    
## Подготовка    
1. Создать свою ветку `feature\<Ваша фамилия и инициалы>` от ветки main в репозитории [https://github.com/AllikarDD/lab3/tree/main](https://github.com/AllikarDD/lab3/tree/main)   
2. Нам потребуются контейнеры из Лабораторной 1, но нужно будет изменить dockerfile, который мы создавали в ней   
    2.1 Добавить строчку в dockerfile    
    `RUN apt-get update && apt-get install -y ansible && apt-get install -y docker-compose-plugin`   
    Добавляем `ansible ` и `docker-compose`   
    Итоговый dockerfile должен выглядеть вот так    
    ```
FROM jenkins/jenkins:2.462.2-jdk17
USER root
RUN apt-get update && apt-get install -y lsb-release
RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc \
  https://download.docker.com/linux/debian/gpg
RUN echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
  https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
RUN apt-get update && apt-get install -y docker-ce-cli
RUN apt-get update && apt-get install -y ansible && apt-get install -y docker-compose-plugin
USER jenkins
RUN jenkins-plugin-cli --plugins "blueocean docker-workflow"	

```
    2.2 Выполняем команду    
    ```
docker build -t myjenkins-blueocean:2.462.2-1 .
```
    2.3 Запускаем контейнер    
    ```
docker run --name jenkins-blueocean --restart=on-failure --detach ^
  --network jenkins --env DOCKER_HOST=tcp://docker:2376 ^
  --env DOCKER_CERT_PATH=/certs/client --env DOCKER_TLS_VERIFY=1 ^
  --volume jenkins-data:/var/jenkins_home ^
  --volume jenkins-docker-certs:/certs/client:ro ^
  --publish 8080:8080 --publish 50000:50000 myjenkins-blueocean:2.462.2-1
```
       
   
3 Заходим в Jenkins по [http://localhost:8080/](http://localhost:8080/) и создаем новый pipeline "Ansible"   
![image.png](files\image_9.png)    
4 В Pipeline scripts вставляем код и запускам pipeline   
```
pipeline { // задаем тон groovy, даем понять что здесь у нас шаги пайплайна
    agent any // если у нас есть какие то jenkins агенты специально для Ansible мы можем их указать здесь, у меня только один агент, поэтому я поставлю any
    
    stages { // здесь определяем какие шаги в пайплайне
        stage('Deploy') { // наш деплой
            steps { // определяем шаги уже самого стейджа
                sh 'ansible --version' // выводим версию Ansible, команда sh просто выполнит скрипт из консоли
            }
        }
    }
}
```
Ожидаемый результат:   
![image.png](files\image.png)    
## Задание 1.  Настроить подключение к Git из Jenkins   
   
1. Создайте еще один pipeline c названием task1   
![image.png](files\image_8.png)    
   
2 В Pipeline scripts вставляем код и запускам pipeline, смотрим на результаты билда   
```
pipeline {
    agent any

    stages {
        stage('Checkout') { 
            steps {
                git branch: 'main', url: "https://github.com/AllikarDD/lab3"
            }
        }
        stage('Deploy') {
            steps {
                sh 'ls' 
            }
        }
    }
```
![image.png](files\image_h.png)    
3. Фикс авторизации   
  3.1   
  Jenkins провалился на первом этапе, не смог скачать исходники с Git, потому что не смог установить соединение с Github. Ответ на вопрос  «как это пофиксить?» в тексте ошибки. Мы увидим эту ошибку, если соединимся по ssh с локальным ssh-agent, но при этом сервера нет в known\_hosts. Добавим сервер Github для Jenkins.   
  Логинимся на машину с Jenkins, от лица пользователя Jenkins, если вы не переставляли пользователя для Jenkins и запускаем ssh github.com. На вопрос «Do you want to add gihutb.com to known hosts» жмем Y. Первая ошибка больше не будет мешать.   
  3.2 Запускаем пайплайн второй раз:   
  Ошибка в stderr поменялась на Permission Denied — для подключения по ssh к Github нужен ключ, который Github знает. Добавим этот ключ.   
  Убеждаемся, что плагин Credentials установлен: Dashboard → Manage Jenkins → Manage Plugins, если нет, то установим его.   
  3.3   
  Генерируем ключ для Github и добавляем его в аккаунт.   
  Запустим команду на машине с linux   
  ```
Ssh-keygen -t rsa

```
  После ответа на вопросы об имени ключа (укажите другое имя кроме id\_rsa, если ваш стандартный ключ уже используется, поскольку его можно случайно перезаписать), защита паролем (выбираем «no») вы получите пару приватный/публичный ключ. Публичный ключ имеет расширение .pub после имени, приватный ключ его не имеет.   
  **Важно!** Старайтесь не светить свой приватный ключ, особенно если он используется где то для доступа к защищенным репозиториям, это вопрос безопасности.   
  После генерации ключей идем на github.com, кликаем на иконку профиля справа, Settings→SSH and GPG keys и попадаем на страницу добавления ssh ключей профиля.   
  3.4 New SSH Key   
  Копируем **ПУБЛИЧНУЮ** часть ключа и называем его произвольным именем   
  3.5 В Dashboard → Manage Jenkins → Credentials нажимаем на Global и Add Credentials.   
  В открывшемся окне выбираем SSH Username with private key и заполняем данные.   
  ID  — идентификатор, уникальный для каждого credential, по которому вызывают их из пайплайна. Username — имя для вашего профиля Github. Private key → нажмите Enter Directly → Add и скопируйте значение из сгенерированного **Приватного** ключа.   
  ![image.png](files\image_g.png)    
  3.6 Меняем скрипт, добавляем credentialsId   
  ```
pipeline {
    agent any

    stages {
        stage('Checkout') { 
            steps {
                git branch: 'main', url: "https://github.com/AllikarDD/lab3", credentialsId: 'github_key' // здесь добавляем credentialsId и указываем наш ID. 
            }
        }
        stage('Deploy') {
            steps {
                sh 'ls' 
            }
        }
    }
```
Ожидаемый ответ   
![image.png](files\image_f.png)    
## Задание 2    
1. Создайте еще один pipeline c названием task2   
   
2 В Pipeline scripts вставляем код, где указываем вашу ветку созданную ранее `feature\<Ваша фамилия и инициалы>` и ваш credentialsId   
```
pipeline {
    agent any

    stages {
        stage('Checkout') { 
            steps {
                git branch: '', url: "https://github.com/AllikarDD/lab3", credentialsId: 'github_key' // здесь добавляем credentialsId и указываем наш ID. 
            }
        }
        stage('Deploy1') {
            steps {
                sh 'docker build -t lab-agent .' 
            }
        }
        stage('Deploy2') {
            steps {
                sh 'docker compose up -d' 
            }
        }
        stage('Deploy3') {
            steps {
                sh 'docker exec -t task2-ansible-1 ansible all -m ping' 
            }
        }
        stage('Deploy4') {
            steps {
                sh 'echo END' 
            }
        }
    }
}
```
3. У вас будет ошибка пингов, исправьте ёё (Подсказка: нужно внести изменения в вашей ветке)   
   
## Задание 3   
   
1. Создайте еще один pipeline c названием task3   
2. Ставим параметраризируемую сборку и добавляем параметр BRANCH   
![image.png](files\image_0.png)    
   
2 В Pipeline scripts вставляем код, добавляем созданный параметр   
```
pipeline {
    agent any

    stages {
        stage('Checkout') { 
            steps {
                git branch: "${BRANCH}", url: "https://github.com/AllikarDD/lab3", credentialsId: 'github_key' // здесь добавляем credentialsId и указываем наш ID. 
            }
        }
        stage('Deploy1') {
            steps {
                sh 'docker build -t lab-agent .' 
            }
        }
        stage('Deploy2') {
            steps {
                sh 'docker compose up -d' 
            }
        }
        stage('Deploy3') {
            steps {
                sh 'docker exec -t test3-ansible-1 ansible all -m ping' 
            }
        }
        stage('Deploy4') {
            steps {
                sh 'echo END' 
            }
        }
    }
}

```
3. Запустить с параметром вашей ветки    
![image.png](files\image_w.png)    
   
## Задание 4    
1. Создайте еще один pipeline c названием task4   
2. Создать параметр playbook    
3. Параметраризировать запуск различных ansible-playbook    
4. Попробуйте запустить /playbooks/playbook-database.yml  через этот параметр    
   
   
