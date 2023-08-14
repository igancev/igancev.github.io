---
title: gRPC сервер и клиент на Symfony
excerpt: Поднимем gRPC сервер и сгенерируем к нему gRPC клиент, в рамках Symfony Framework
keywords: php, grpc, grpc server, roadrunner, symfony framework, docker, grpc сервер, pfotobuf, protoc, php-grpc-plugin
key: grpc-server-on-symfony
tags: symfony grpc php roadrunner docker
slug: grpc-server-on-symfony
mainImage: /assets/articles/symfony-grpc/symfony-grpc.png
---

![symfony + grpc = love](/assets/articles/symfony-grpc/symfony-grpc.png)

Поднимем gRPC сервер и сгенерируем к нему gRPC клиент, в рамках Symfony Framework

Этот гайд - своего рода продолжение предыдущей статьи про [gRPC сервер на Spiral](/2023-05-08-grpc-server-on-php)
{:.info}

## Вступление

О том, что такое gRPC, и почему нам нужно для gRPC сервера на PHP использовать `Roadrunner`, я рассказывал
[тут](/2023-05-08-grpc-server-on-php#что-такое-grpc).

❤️  Эти киты посвящаются [всем проголосовавшим](/2023-05-08-grpc-server-on-php#планы) в прошлом посте [в телеграме](https://t.me/igancev_ru/18) за то, чтобы этот мануал о **Symfony + gRPC** вышел в свет: 🐳🐳🐳🐳🐳🐳🐳🐳🐳🐳🐳🐳🐳🐳🐳🐳🐳🐳🐳🐳🐳🐳🐳🐳🐳🐳🐳🐳🐳🐳🐳🐳🐳🐳🐳.
Ставь хлопки в ладоши 👏 к этому [посту в телеграме](https://t.me/igancev_ru/19), если обрадовался выходу материала! 
{:.info}

## Постановка задачи и план

Наша цель - научиться использовать gRPC совместно с PHP фреймворком Symfony. Исходя из этого, мы:

- Придумаем задачу, которую в процессе будем решать
- Разработаем **.proto** контракт
- Установим **symfony**
- Настроим **docker окружение** для приложения
- Установим **roadrunner-bundle**
- **Сгенерируем php код** с заглушками gRPC сервера на основе .proto контракта
- **Протестируем** сервер сторонним gRPC клиентом
- Сгенерируем **клиентский php код** и протестируем его 

## Подготовка

В качестве учебной задачи давайте разработаем несложный сервис _"Укорачиватель url адресов"_. Укорачиватель
на вход будет принимать длинную ссылку, а возвращать короткую.

![Наглядная схема взаимодействия клиента и сервера](/assets/articles/symfony-grpc/shortener-schema.png)

### .proto контракт

```protobuf
syntax = "proto3";

package grpc.shortener;

message LongUrl {
  string url = 1;
}

message ShortUrl {
  string url = 1;
}

service UrlShortener {
  rpc shorten (LongUrl) returns (ShortUrl) {
  }
}
```

### Установка symfony

```shell
docker run -v `pwd`:/app -u $(id -u):$(id -g) \
  composer create-project symfony/skeleton url-shortener
```

### Настройка docker окружения для приложения

Создадим следующий `Dockerfile`:

```dockerfile
# ./deployment/docker/roadrunner/Dockerfile

FROM php:8.2-cli-alpine

WORKDIR /app

COPY --from=mlocati/php-extension-installer /usr/bin/install-php-extensions /usr/bin/

RUN install-php-extensions bcmath intl opcache zip sockets grpc protobuf

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

CMD ["/app/bin/rr", "serve", "-d", "-c", "/app/.rr.yaml"]
```

И следующий `docker-compose.yml`:

```yaml
# ./docker-compose.yml
version: '2'

services:
  rr:
    build:
      dockerfile: ./deployment/docker/roadrunner/Dockerfile
    ports:
      - '9001:9001' # шарим порт 9001
    user: '1000:1000' # работа от текущего юзера в системе
    volumes:
      - ./:/app # монтируем проект в /app
```

Пока просто соберем образ, без запуска контейнера:

```bash
docker-compose build
```

### Установка roadrunner-bundle

Бандл [baldinof/roadrunner-bundle](https://github.com/Baldinof/roadrunner-bundle) предоставляет готовый _RoadRunner Worker под Symfony_, ряд конфигов и настроек.
Помогает быстро стартовать на symfony в связке с Roadrunner. Установим его и согласимся с установкой рецепта (**y**).

```bash
docker-compose run rr composer req baldinof/roadrunner-bundle
```

В результате установки рецепта был зарегистрирован бандл и скопированы некоторые конфигурационные файлы.

Отредактируем yaml конфиги roadrunner `.rr.yaml` и `.rr.dev.yaml`: 
- удалим ненужный нам плагин `http` сервера:
    ```yaml
  http:
    address: 0.0.0.0:8080
    ... # удаляем всю секцию
    ```
  
- добавим вместо него секцию `grpc`:
  ```yaml
  grpc:
    listen: "tcp://0.0.0.0:9001"
    proto:
      - "proto/shortener.proto"
    ```

Поместим ранее написанный `.proto` контракт в файл `./proto/shortener.proto`.

Установим пакеты [grpc/grpc](https://github.com/grpc/grpc-php) и [spiral/roadrunner-grpc](https://github.com/roadrunner-php/grpc):

```bash
docker-compose run rr composer req grpc/grpc spiral/roadrunner-grpc
````

Понадобится бинарник Roadrunner:

```bash
docker-compose run rr composer req --dev spiral/roadrunner-cli \
  && ./vendor/bin/rr get-binary -l ./bin
```

## Кодогенерация

Нам нужно генерировать PHP код _сервера_ и _клиента_ gRPC. За генерацию отвечает утилита `protoc`, которой требуется плагин
для генерации кода (`protoc-gen-php-grpc`).

Для генерации **серверного кода** воспользуемся
[плагином от проекта Roadrunner](https://github.com/roadrunner-server/roadrunner/releases) `protoc-gen-php-grpc`.
Но, к сожалению, он не генерирует код классов клиента. Хоть их и несложно написать руками, нам бы хотелось
максимальную автоматизацию. 

Поэтому для генерации **клиентского кода** используем 
[официальный плагин](https://grpc.io/docs/languages/php/basics/#setup) `grpc_php_plugin`,
который нужно собрать из исходников руками.

Инструкции по сборке `protoc` и `grpc_php_plugin`
[есть в документации](https://grpc.io/docs/languages/php/basics/#setup). Но, процесс сборки может быть муторным
из-за разрешения зависимостей. Поэтому я [подготовил docker образ](https://github.com/igancev/protoc-php-plugins),
в котором производится сборка `protoc`, `grpc_php_plugin`,
и получение `protoc-gen-php-grpc`. Остается лишь воспользоваться готовеньким 🙂.

Будем складывать сгенерированный код в директорию `generated`. Создадим ее:

```bash
mkdir ./generated
```

Серверный и клиентский код будем генерировать одной и той же командой, будет лишь отличаться значение аргумента `--plugin`.

### Серверный код

Для генерации серверных заглушек используем плагин `/usr/bin/protoc-gen-php-grpc`:

```bash
docker run -u $(id -u):$(id -g) -v `pwd`:/app \
  ghcr.io/igancev/protoc-php-plugins:latest protoc \
  --plugin=protoc-gen-grpc=/usr/bin/protoc-gen-php-grpc \
  --php_out=./generated \
  --grpc_out=./generated \
  ./proto/shortener.proto
```

Как результат, в директории `generated` мы увидим следующий набор файлов:

```
GPBMetadata
  - Proto
    - Shortener.php
Grpc
  - Shortener
    - LongUrl.php
    - ShortUrl.php
    - UrlShortenerInterface.php
```

Где:

- `LongUrl` и `ShortUrl` - классы, отражающие DTO нашего proto контракта.
- `UrlShortenerInterface` - интерфейс сервиса, который нам **нужно будет реализовать**, поместив туда логику приложения.

Реализуем `UrlShortenerInterface`:

```php
<?php
// ./src/UrlShortenerService.php

declare(strict_types=1);

namespace App;

use Grpc\Shortener\LongUrl;
use Grpc\Shortener\ShortUrl;
use Grpc\Shortener\UrlShortenerInterface;
use Spiral\RoadRunner\GRPC;

final class UrlShortenerService implements UrlShortenerInterface
{
    public function shorten(GRPC\ContextInterface $ctx, LongUrl $in): ShortUrl
    {
        $longUrl = $in->getUrl();
        $shortedUrl = $this->buildShortUrl($longUrl);

        return (new ShortUrl())->setUrl($shortedUrl);
    }

    private function buildShortUrl(string $longUrl): string
    {
        // Вместо настоящей реализации мы имитируем бурную деятельность
        $randomStr = bin2hex(\random_bytes(5));

        return 'https://examp.le/' . $randomStr;
    }
}
```

### Клиентский код

Для генерации клиента используем плагин `/usr/bin/grpc_php_plugin`:

```bash
docker run -u $(id -u):$(id -g) -v `pwd`:/app \
  ghcr.io/igancev/protoc-php-plugins:latest protoc \
  --plugin=protoc-gen-grpc=/usr/bin/grpc_php_plugin \
  --php_out=./generated \
  --grpc_out=./generated \
  ./proto/shortener.proto
```

На выходе появится лишь один новый класс `UrlShortenerClient`, это и есть реализация gRPC клиента:

```
Grpc
  - Shortener
    - UrlShortenerClient.php
```

### composer.json autoload

Для автозагрузки сгенерированных файлов, добавим следующие строки в секцию `autoload.psr-4` файла `composer.json`:

```json
...
"autoload": {
  "psr-4": {
    ...
    "GPBMetadata\\": "generated/GPBMetadata",
    "Grpc\\": "generated/Grpc"
  }
},
...
```

Применим изменения:

```bash
docker-compose run rr composer du
```



## Запуск

```bash
docker-compose up
```

Все хорошо, если в выводе нет ошибок, и есть примерно такой текст `grpc server was started` 🚀.

### Тестируем сервер сторонним клиентом

Можно тестировать любым удобным gRPC клиентом, а я в этот раз воспользуюсь `grpcurl`:

```bash
grpcurl -plaintext -d '{"url":"https://example.com/long-url"}' \
    -import-path ./proto -proto shortener.proto \
    localhost:9001 grpc.shortener.UrlShortener/shorten
```

Успех! 🥳 Мы получили следующий результат:

```
{
  "url": "https://examp.le/bc427772f9"
}
```

### Тестируем взаимодействие php клиента с сервером

Создадим консольную команду, из которой будем вызывать клиентский код. Результат будем выводить в консоль:

```php
<?php
// ./src/ShortenUrlCliCommand.php

declare(strict_types=1);

namespace App;

use Grpc\ChannelCredentials;
use Grpc\Shortener\LongUrl;
use Grpc\Shortener\ShortUrl;
use Grpc\Shortener\UrlShortenerClient;
use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

#[AsCommand('shortener:shorten', 'Shorten url by shortener grpc service')]
final class ShortenUrlCliCommand extends Command
{
    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $urlShortenerClient = new UrlShortenerClient('127.0.0.1:9001', [
            'credentials' => ChannelCredentials::createInsecure(),
        ]);

        $longUrl = (new LongUrl())->setUrl('https://example.com/long-url');

        /** @var ShortUrl $shortUrl */
        [$shortUrl] =  $urlShortenerClient->shorten($longUrl)->wait();

        $output->writeln('Short url: ' . $shortUrl->getUrl());

        return Command::SUCCESS;
    }
}
```

Запустим команду:

```bash
docker-compose exec rr bin/console shortener:shorten
```

Результат:

```
Short url: https://examp.le/29197d611f
```

👍 👏 💪

## Заключение

Сегодня мы подружили Symfony и gRPC, реализовав простую, демонстрационную задачку. И мы молодцы. 

## Комментарии

Если есть желание обсудить эту тему, то сделать это можно под [постом данной статьи в
телеграм канале](https://t.me/igancev_ru/19), на который также можно подписаться, чтобы не пропустить новые
материалы. Всем добра! 🤚