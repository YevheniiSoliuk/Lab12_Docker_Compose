# Lab12_Docker_Compose
## 1. Zawartość plików Dockerfile

**Dockerfile dla apacha**

```docker
FROM httpd:2.4-alpine
COPY apache.conf /usr/local/apache2/conf/apache.conf
RUN echo "Include /usr/local/apache2/conf/apache.conf" \
    >> /usr/local/apache2/conf/httpd.conf
```

Ten Dockerfile jest do utworzenia obrazu httpd na bazie alpine, w którym do standardowej konfiguracji apacha będzie dodana jeszcze konfiguracja własna.

**Dockerfile dla PHP**

```docker
FROM php:7.2.7-fpm-alpine3.7
```

Podany Dockerfile będzie wykorzystywany do utworzenia obrazu z PHP-FPM dla szybszego działania na bazie alpine, jak poprzednio. 

## 2. Zawartość innych plików

Plik konfiguracyjny dla serwera apache ***apache.conf***

```
ServerName localhost

LoadModule deflate_module /usr/local/apache2/modules/mod_deflate.so
LoadModule proxy_module /usr/local/apache2/modules/mod_proxy.so
LoadModule proxy_fcgi_module /usr/local/apache2/modules/mod_proxy_fcgi.so

<VirtualHost *:80>
   
    ProxyPassMatch ^/(.*\.php(/.*)?)$ fcgi://php:9000/var/www/html/$1
    DocumentRoot /var/www/html/
    <Directory /var/www/html/>
        DirectoryIndex index.php
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
    
    CustomLog /proc/self/fd/1 common
    ErrorLog /proc/self/fd/2
</VirtualHost>
```

W tym pliku konfiguracyjnym jest utworzenia hostu wirtualnego oraz połączenia proxy dla możliwości przesyłania plików php do serwera apacha. Do tego jeszcze jest zmieniona poczatkowa dyrektoria plików z ustawieniem pliku index.php, jako pliku domowego przy starcie serweru.

Zawartość pliku ***index.php***

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Document</title>
</head>
<body>
  <h1>Docker Compose</h1>
  <?php 
    phpinfo();
  ?>
</body>
</html>
```

W pliku index.php znajdują się zwykły nagłówek oraz wynik funkcji *phpinfo().*

Zawartość pliku ze zmiennymi środowyskowymi ***.env***

```
MYSQL_ROOT_PASS=rootpass
MYSQL_DATABASE=docker_database
```

Został utworzony plik *.env* dla podalszego wykorzystania zmiennych w konfiguracji docker-compose.yml przy ustawieniu zmiennych środowiskowych dla mysql oraz phpmyadmin.

## 3. Zawartość pliku docker-compose.yml

```yaml
version: '3.8'

services:
  apache:
    build: "./apache/"
    ports: 
      - "8080:80"
    volumes:
      - ./public_html/:/var/www/html/
    depends_on:
      - php
      - mysql
    restart: unless-stopped
    networks:
      - frontend
      - backend

  php:
    build: "./php/"
    volumes:
      - ./public_html/:/var/www/html/
    networks:
      - backend
  
  mysql:
    image: mysql:latest
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASS}
    networks:
      - frontend
  
  phpmyadmin:
    image: phpmyadmin/phpmyadmin
    ports:
      - '8000:80'
    restart: always
    environment:
      PMA_HOST: mysql
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASS}
    links:
      - mysql
    depends_on:
      - mysql
    networks:
      - frontend

networks:
  frontend:
  backend:
```

W pliku docker-compose.yml utworzone cztery serwisy, czyli przyszłe kontenery, z dwoma sieciami. Serwisy apache oraz php korzystają ze wspólnego wolumenu oraz sieci **backend**, żeby przesyłać i odbierać pliki z wolumenu. Natomiast serwisy mysql oraz phpmyadmin połączeni z siecią **frontend**, ponieważ ostatni jest graficzną reprezentacją bazy danych, zatem musi odbierać dane z bazy. Dla otrzymania dostępu do phpmyadmin do serwisów były dodane zmienne środowiskowe okreslające nazwę bazy danych, oraz hasło dla root-a.

## 4. Wygenerowana reprezentacja graficzna pliku docker-compose.yml
![docker-compose](https://user-images.githubusercontent.com/78736395/172062638-07bdbece-cb51-4dc9-b49f-407d728f5318.png)

## 5. Wynik działania
Po wykonaniu polecenia
```
docker compose up -d
```
kontenery zostały uruchomione w kolejności, która została podana w pliku *docker-compose.yml*
![Zrzut ekranu 2022-06-05 193019](https://user-images.githubusercontent.com/78736395/172062839-57b6b6aa-e93b-4d4c-b018-f2d26300cdc7.jpg)

![Zrzut ekranu 2022-06-05 193143](https://user-images.githubusercontent.com/78736395/172062896-a11bfe3a-e2d2-413b-9828-f8756d7fc0a8.jpg)

Po uruchomieniu można uzyskać dostęp do strony za adresą localhost:8080 i otrzymać taki wynik
![Zrzut ekranu 2022-06-05 182506](https://user-images.githubusercontent.com/78736395/172062973-ac4e72e6-a2af-4647-aba3-dc4b18fad32c.jpg)

Aby uzyskać dostęp do phpmyadmin potrzebowane jest tylko przejście na strone localhost:8000 oraz wpisania loginu, jako **root** oraz hasła podanego w pliku ***.env***
![Zrzut ekranu 2022-06-05 182438](https://user-images.githubusercontent.com/78736395/172063046-3edfd1ad-55e5-460a-8355-94359f5f4727.jpg)

W wyniku jest otrzymany dostęp do bazy danych **docker**
![Zrzut ekranu 2022-06-05 182321](https://user-images.githubusercontent.com/78736395/172063074-a4857775-c921-432d-871f-3893a30dbc89.jpg)
