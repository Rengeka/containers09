# Лабораторная работа №9: Оптимизация образов контейнеров

## Цель работы
Целью работы является знакомство с методами оптимизации образов.

## Задание
Сравнить различные методы оптимизации образов:

    Удаление неиспользуемых зависимостей и временных файлов
    Уменьшение количества слоев
    Минимальный базовый образ
    Перепаковка образа
    Использование всех методов

## Выполнение

Создаём github репозиторий containers09

Внутри создаём директорию ./site и внутрь добавляем html файл index.html, представляющий наш сайт

Создаём Dockerfile.raw со следующим содержанием:

```dockerfile
# create from ubuntu image
FROM ubuntu:latest

# update system
RUN apt-get update && apt-get upgrade -y

# install nginx
RUN apt-get install -y nginx

# copy site
COPY site /var/www/html

# expose port 80
EXPOSE 80

# run nginx
CMD ["nginx", "-g", "daemon off;"]
```

Создаём образ на основе этого докерфайла
```docker 
docker image build -t mynginx:raw -f Dockerfile.raw .
```

Создаём Dockerfile.clean и добавляем внутрь 

```Dockerfile
# create from ubuntu image
FROM ubuntu:latest

# update system
RUN apt-get update && apt-get upgrade -y

# install nginx
RUN apt-get install -y nginx

# remove apt cache
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# copy site
COPY site /var/www/html

# expose port 80
EXPOSE 80

# run nginx
CMD ["nginx", "-g", "daemon off;"]
```

Создаём образ на основе второго докерфайла и смотрим список образов
```docker 
docker image build -t mynginx:clean -f Dockerfile.clean .
docker image list
```

![Alt text](/images/Снимок%20экрана%202025-04-20%20123106.png "image")

Как мы видим не смотря на то, что в новом докерфайле мы удаляем ненужные директории, т.к мы создаём дополнительный промежуточный образ, сам image всё равно весит столько же

Создаём Dockerfile.few и уменьшаем в нём количество слоёв

```Dockerfile
# create from ubuntu image
FROM ubuntu:latest

# update system
RUN apt-get update && apt-get upgrade -y && \
    apt-get install -y nginx && \
    apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# copy site
COPY site /var/www/html

# expose port 80
EXPOSE 80

# run nginx
CMD ["nginx", "-g", "daemon off;"]
```

Создаём образ на основе докерфайла .few и смотрим список образов
```docker 
docker image build -t mynginx:few -f Dockerfile.few .
docker image list
```

![Alt text](/images/Снимок%20экрана%202025-04-20%20123919.png "image")

Как мы вмидим новый образ сэкономил нам почти 50 мегабайт

Создадим Dockerfile.alpine на основе более легковесного образа, чем ubuntu

```Dockerfile
# create from alpine image
FROM alpine:latest

# update system
RUN apk update && apk upgrade

# install nginx
RUN apk add nginx

# copy site
COPY site /var/www/html

# expose port 80
EXPOSE 80

# run nginx
CMD ["nginx", "-g", "daemon off;"]
```

Создаём образ на основе докерфайла .alpine и смотрим список образов
```docker 
docker image build -t mynginx:alpine -f Dockerfile.alpine .
docker image list
```

![Alt text](/images/Снимок%20экрана%202025-04-20%20124033.png "image")

Теперь мы экономим более 100 мегабайт по сравнению с предыдущим образом

Попробуем воспользоваться перепаковкой. Возьмём наш исходный .raw образ и перепакуем его, чтобы избавиться от слоёв

```docker
docker container create --name mynginx mynginx:raw
docker container export mynginx | docker image import - mynginx:repack
docker container rm mynginx
docker image list
```

![Alt text](/images/Снимок%20экрана%202025-04-20%20125959.png "image")

Как видно, даже самый тяжеловесный образ после перепоквки стал весить на 40 мегабайт меньше.

Теперь перепауем наш легковесный образ. Создадим Dockerfile.min

```Dockerfile
# create from alpine image
FROM alpine:latest

# update system, install nginx and clean
RUN apk update && apk upgrade && \
    apk add nginx && \
    rm -rf /var/cache/apk/*

# copy site
COPY site /var/www/html

# expose port 80
EXPOSE 80

# run nginx
CMD ["nginx", "-g", "daemon off;"]
```

Сбилдим и перепакуем

```docker
docker image build -t mynginx:minx -f Dockerfile.min .
docker container create --name mynginx mynginx:minx
docker container export mynginx | docker image import - myngin:min
docker container rm mynginx
docker image list
```

![Alt text](/images/Снимок%20экрана%202025-04-20%20130823.png "image")

Как мы видим нам удалось добиться размера в 9.6 мегабайт вместо первоначальных 173

Таблицу всех образов :

| REPOSITORY | TAG    | SIZE   |
|------------|--------|--------|
| mynginx    | min    | 9.26MB |
| mynginx    | minx   | 9.28MB |
| mynginx    | repack | 133MB  |
| mynginx    | alpine | 11.8MB |
| mynginx    | few    | 125MB  |
| mynginx    | clean  | 173MB  |
| mynginx    | raw    | 173MB  |

## Вопросы

### Какой метод оптимизации образов вы считаете наиболее эффективным?

По моему мнению самым эффективным способом тут оказался способ созщдания образа на основе уже легковестного. Мы сумели сэкономить более 100 мегабайт. Однако ключевым аспектом в оптимизации образов я всётаки считаё уменьшение количества слоёв и перепакову.

### Почему очистка кэша пакетов в отдельном слое не уменьшает размер образа?

Слои неизменяемы. То есть даже если мы удалим из последнего слоя всё, предыдущие не перестанут весить меньше

### Что такое перепаковка образа?

Это экспорт файловой системы из контейнера и её импорт в новый образ. Это позволяет собрать полский образ без промежуточных слоёв