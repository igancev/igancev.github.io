---
title: gRPC сервер на PHP с помощью Roadrunner, Spiral Framework, Docker
excerpt: Разберемся как приготовить gRPC сервер на PHP при том, что из коробки gRPC не предоставляет сервер под этот язык. В этом нам поможет небезызвестный Roadrunner, способный взять часть задач на себя
keywords: php, grpc, grpc server, roadrunner, spiral framework, docker, grpc сервер, xdebug
key: grpc-server-on-php
tags: php grpc roadrunner spiral docker xdebug
slug: grpc-server-on-php
mainImage: /assets/articles/php-grpc-server/php-grpc-server.png
---

![Логотипы php, spiral, grpc, roadrunner и docker](/assets/articles/php-grpc-server/php-grpc-server.png)

Разберемся как приготовить gRPC сервер на PHP при том, что из коробки gRPC не предоставляет сервер под этот язык.
В этом нам поможет небезызвестный Roadrunner, способный взять часть задач на себя

Есть еще другая статья по теме, где пишем [gRPC сервер на Symfony](/2023-08-14-grpc-server-on-symfony).
{:.info}

## Предисловие

### Что такое gRPC

[gRPC](https://grpc.io) - Кроссплатформенный фреймворк удаленного вызова процедур (RPC - Remote Procedure Calls),
разработанный Google. Позволяет:

- Декларативно описать контракты взаимодействия на языке [Protocol Buffers](https://protobuf.dev)
- Сгенерировать по описанным контрактам клиентский и частично серверный код любого поддерживаемого языка
  (в том числе и PHP)
- Эффективно передавать данные по сети благодаря компактной сериализации

### gRPC сервер на PHP?

Проблема в том, что **из коробки на PHP нет реализации gRPC сервера**. 
В [официальной документации](https://grpc.io/docs/languages/php/basics) сказано, что на PHP
можно создать только клиент под GRPC, а для сервера стоит использовать другой язык.

`Also note that currently, you can only create clients in PHP for gRPC services.
Use another language to create a gRPC server.`

Причины при желании можно подробно изучить в [данном топике](https://groups.google.com/g/grpc-io/c/F3IyYaI_6S0/m/Cb55T92pBwAJ).

Чтобы не отказываться от идеи писать серверную логику приложения на PHP в связке с gRPC, нам нужно промежуточное звено, которое взяло бы на себя
всю работу по обеспечению gRPC сервера, конкретно транспортную часть. А данные на обработку передавало бы в PHP. Чтобы вся полезная нагрузка,
бизнес логика, выполнялась на PHP.
Именно по такой схеме работает
`Roadrunner`.

### Roadrunner

[Roadrunner](https://roadrunner.dev) - высокопроизводительный application сервер, балансировщик нагрузки и диспетчер PHP процессов.
Написан на Go. Позволяет эффективно создавать гибридные PHP/Go приложения, комбинируя лучшее из обоих языков. 

![Схема устройства работы Roadrunner](/assets/articles/php-grpc-server/roadrunner-work-schema.png)

Roadrunner создает пул демонизированных PHP процессов. Выступая как `http` или `gRPC` сервер, Roadrunner принимает запрос и 
передает его на обработку одному из поднятых PHP процессов по бинарному протоколу `Goridge`.
PHP обрабатывает запрос и возвращает ответ Roadrunner - у.

Процесс PHP при этом "не умирает", как это привычно происходит в классической схеме `Nginx + Php-fpm`, а тут же принимает следующий
запрос, не тратя повторно время на инициализацию. Это само по себе значительно **улучшает производительность** PHP. Платой 
за прирост производительности выступает необходимость следить за возможными утечками памяти, которыми зачастую пренебрегают из-за природы
PHP.

Разработка ведется в Белорусской компании **Spiral Scout**, которая также известна и другими своими превосходными open source
разработками `Cycle ORM` и `Spiral Framework`.

### Аналоги

Достойных аналогов, предоставляющих gRPC сервер для PHP,
кроме [Open Swoole](https://openswoole.com/docs/modules/grpc-server) я не нашел. Проект выглядит интересным,
но опыта с ним почти не имел. Начинать не хочется только потому, что в нем
[нет поддержки Xdebug](https://openswoole.com/docs/get-started-swoole#incompatibilities).

## Реализация gRPC сервера

### Цели и постановка задачи

Наша цель - получить навык разработки gRPC сервера на PHP.

Для ее достижения выдумаем и реализуем простенький демо сервис, предоставляющий информацию о курсах валют.
Любой клиент средствами gRPC должен иметь возможность запрашивать текущий курс интересующей валюты у сервера. 

Сервер в свою очередь будет по http или иным образом ходить в источник курсов валют (какая - нибудь биржа или сервис), 
и возвращать актуальный курс на момент времени по запрошенной валютной паре.

![Схема работы демо сервиса курсов валют. Клиент обращается к серверу по grpc, сервер обращается к источнику валют по http, и ответ возвращается обратно к клиенту по цепочке](/assets/articles/php-grpc-server/demo-service-schema.png)

Я произвожу все действия на ПК с дистрибутивом `Manjaro Linux` на борту. Думаю все можно повторить и на MacOS/Windows, 
но скорее всего с оговорками, которые в данной статье затронуты не будут.

### Установка Spiral

Начнем с установки PHP фреймворка [Spiral](https://spiral.dev). 

```bash
composer create-project spiral/app currency-rate
```

На первых шагах обходимся локально установленным `php` и `composer`, чтобы пройти консольный мастер установки от Spiral,
настроив скелет приложения. Далее все завернем в `docker`, как положено.

На вопрос: "Какой пресет приложения мы хотим установить?" отвечаем вариант 3, `gRPC`

```bash
Which application preset do you want to install?
  [1] Web
  [2] Cli
  [3] gRPC
Make your selection (default: 1): 3
```

На следующих вопросах отказываемся от демо данных и от установки любых компонентов, чтобы на старте не заиметь ничего
лишнего.

После чего установщик любезно выкачает зависимости, скачает плагин `protoc-gen-php-grpc` и бинарник
роадранера `rr`, разместив их в корне.

### Docker окружение

Нам потребуется собрать контейнер с `Roadrunner`. По сути это будет образ на базе контейнера с `php` c нужными
расширениями, с установленными внутри утилитами: 

- `composer`
- `rr` (бинарник Roadrunner - а)
- `protoc` утилита кодогенерации
- `protoc-gen-php-grpc` плагин для утилиты кодогенерации

Создадим в корне проекта директорию `deployment`, где будем хранить всю конфигурацию, связанную с инфраструктурой
развертывания.

```bash
mkdir -p deployment/docker/roadrunner
```

Создадим `Dockerfile` под Roadrunner:

```bash
# ./deployment/docker/roadrunner/Dockerfile

# Базируемся на официальном образе php cli alpine
FROM php:8.2-cli-alpine

# Рабочая директория /app
WORKDIR /app

# Копируем скрипт install-php-extensions от mlocati
# Данный проект рекомендован на официальной странице образов php https://hub.docker.com/_/php
COPY --from=mlocati/php-extension-installer /usr/bin/install-php-extensions /usr/bin/

# Устанавливаем необходимые расширения
RUN install-php-extensions opcache zip sockets \
    protobuf \
    xdebug \
    # Устанавливаем утилиту protoc
    && apk update && apk add --no-cache make protobuf-dev

# Копируем composer
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

# Команда и аргументы запуска
# Используем бинарник Roadrunner и конфиг из корня проекта
# Корень проекта будет смонтирован при запуске в /app
CMD ["/app/rr", "serve", "-d", "-c", "/app/.rr.yaml"]

```

Создадим `docker-compose.yml` в корне проекта:

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

Пока просто соберем образ, без запуска:

```bash
docker-compose build
````

### Пишем .proto контракты

Спроектируем контракты для нашего выдуманного сервиса получения курсов валют.

В корне проекта создадим директорию `proto`, в которой и разместим наши `.proto` файлы:

```protobuf
// ./proto/currency/message.proto
syntax = "proto3";

package currency.dto;

option php_namespace = "GRPCContracts\\Currency";
option php_metadata_namespace = "GRPCContracts\\Currency\\GPBMetadata";

message GetRateRequest {
  string ticker = 1;
}

message CurrencyRateResponse {
  string ticker = 1;
  string rate = 2;
}
```

```protobuf
// ./proto/currency/service.proto
syntax = "proto3";

package currency;

option php_namespace = "GRPCContracts\\Currency";
option php_metadata_namespace = "GRPCContracts\\Currency\\GPBMetadata";

import "message.proto";

service CurrencyRateService {
  rpc GetRate (currency.dto.GetRateRequest) returns (currency.dto.CurrencyRateResponse) {
  }
}
```

- В `message.proto` описаны DTO запроса `GetRateRequest` и ответа `CurrencyRateResponse`.
- В `service.proto` описан сервис `CurrencyRateService` c методом получения курса `GetRate`.
  В качестве аргумента метод принимает описанный выше тип запроса, а возвращает тип ответа. 
  Для их использования в файле производится импорт соседнего `message.proto`.

### Кодогенерация

Сгенерируем php код по написанным proto контрактам.

Внесем изменения в конфигурационном файле Spiral фреймворка
`./app/config/grpc.php`. 

- Ключам массива `namespace` и `servicesBasePath` установим значение `null`
- В массив `services` добавим путь к нашему `service.proto`

```php
<?php
// ./app/config/grpc.php
declare(strict_types=1);

return [
    'binaryPath' => directory('root') . 'protoc-gen-php-grpc',
    'generatedPath' => directory('root') . '/generated',
    'namespace' => null, // тут ставим null
    'services' => [
        // сюда прописываем путь до нашего контракта сервиса
        directory('root') . '/proto/currency/service.proto'
    ],
    'servicesBasePath' => null, // тут ставим null
];
```

И запустим консольную команду генерации:

```bash
docker-compose run rr php app.php grpc:generate
```

В корне проекта обнаружим директорию `generated`, в которой находится
сгенерированный код.

### composer.json

Добавим директорию `generated/GRPCContracts` в автолоад composer - а. Для этого в 
`composer.json` в секцию `autoload.psr-4` добавим строку:

`"GRPCContracts\\": "generated/GRPCContracts"`

Получится примерно следующее:

```json
...
"autoload": {
  "psr-4": {
    "App\\": "app/src",
    "GRPCContracts\\": "generated/GRPCContracts"
  }
},
...
```

И не забудем применить изменения:

```bash
docker-compose run rr composer du
```

### Конфиг Roadrunner

Внесем изменения в конфиг `.rr.yaml` Roadrunner:

- В секциях `rpc.listen` и `grpc.listen` изменим ip адрес на `0.0.0.0` для того, чтобы все работало из под `docker`
- В секции `grpc.proto` укажем путь к основному `proto` файлу контракта

Итоговый файл `.rr.yaml` будет выглядеть так:

```yaml
# ./.rr.yaml
version: '3'
rpc:
  listen: 'tcp://0.0.0.0:6001'
grpc:
  listen: 'tcp://0.0.0.0:9001'
  proto:
    - ./proto/currency/service.proto
server:
  command: 'php app.php'
  relay: pipes
```

### Реализуем код сервиса

Нам необходимо имплементировать сгенерированный интерфейс `\GRPCContracts\Currency\CurrencyRateServiceInterface.php`:

```php
<?php
// ./app/src/CurrencyRateService.php

declare(strict_types=1);

namespace App;

use GRPCContracts\Currency\CurrencyRateResponse;
use GRPCContracts\Currency\CurrencyRateServiceInterface;
use GRPCContracts\Currency\GetRateRequest;
use Spiral\RoadRunner\GRPC;

final class CurrencyRateService implements CurrencyRateServiceInterface
{
    public function GetRate(GRPC\ContextInterface $ctx, GetRateRequest $in): CurrencyRateResponse
    {
        $response = new CurrencyRateResponse();

        $rate = \random_int(70, 75) . '.00';

        $response->setRate($rate);

        $response->setTicker($in->getTicker());

        return $response;
    }
}

```

Чтобы не усложнять пример, в реальные источники курсов валют мы тут не ходим. Вместо этого значение курса заданной валюты
генерируем рандомно.

В возвращаемый ответ `CurrencyRateResponse` кладем запрашиваемый тикер и его курс.

### Первый запуск

```bash
docker-compose up -d
```

Проверяем, стартанул ли контейнер:

```bash
docker-compose ps
```

В выводе должен находиться наш контейнер, а в колонке `STATUS` не должно быть ошибок. Если в статусе ошибка, или вовсе
контейнер отсутствует, то необходимо анализировать логи `docker-compose logs rr`.

### Первый тест через Postman

Можно использовать любой gRPC клиент. В силу популярности `Postman` как http клиента, возьмем его. Postman,
помимо http, поддерживает и gRPC протокол.

В дропдауне выбора метода необходимо сначала вручную загрузить proto файл `service.proto`, и выбрать директорию, где
лежат остальные `proto` файлы. Это приходится делать потому, что `Roadrunner`, к сожалению, пока еще 
[не поддерживает механизм "рефлексии"](https://github.com/roadrunner-server/grpc/pull/63),
который автоматически вернул бы контракты, без необходимости предоставлять клиенту контракты вручную.
{:.warning}

![Вызов gRPC метода получения курса валют GetRate через Postman](/assets/articles/php-grpc-server/postman.png)

Можно было еще воспользоваться например [grpcox](https://github.com/gusaul/grpcox). Я же предпочитаю в основном этот 
[gRPC плагин на PhpStorm](https://plugins.jetbrains.com/plugin/16889-grpc).

![Вызов gRPC метода получения курса валют GetRate через Phpstorm, плагин phpstorm-grpc-pluggin](/assets/articles/php-grpc-server/phpstorm-grpc-pluggin.png)

## Настройка Xdebug

1. Убедимся, что в `deployment/docker/roadrunner/Dockerfile`, в списке расширений на установку присутствует `xdebug`

2. Создадим файл `deployment/docker/roadrunner/xdebug.ini` со следующим содержимым:
  ```ini
# ./deployment/docker/roadrunner/xdebug.ini
xdebug.mode=debug
xdebug.client_host=172.17.0.1
xdebug.start_with_request=yes
xdebug.discover_client_host=1
xdebug.idekey=PHP_STORM
  ```
3. Примонтируем его в `docker-compose.yml` внутрь контейнера `rr`, секция `services.rr.volumes`:
  ```yaml
volumes:
  - ./deployment/docker/roadrunner/xdebug.ini:/usr/local/etc/php/conf.d/xdebug.ini
  ...
  ```
4. В `docker-compose.yml`, в секцию `services.rr.environment` добавим переменную окружения:
  ```yaml
environment:
  PHP_IDE_CONFIG: 'serverName=rr'
  ...
  ```
5. В конфиг `.rr.yaml` добавим секцию `grpc.pool.num_workers` и установим значение `1`. 
  Добавим секцию `grpc.pool.debug` и установим значение `true`
  ```yaml
grpc:
    ...
    pool:
      num_workers: 1
      debug: true
  ```

Параметр `debug: true` позволяет нормально дебажить код, запущенный в воркере. Если его не указать, то после изменения
кода в файлах вы обнаружите, что брейкпойнты в IDE указывают не на ожидаемые строки. Это происходит потому, что 
воркер был запущен на старой версии кода, до внесения изменений.
Раньше этот же результат достигался с помощью плагина `reload`, который следил за
изменениями файлов и перезапускал воркеры. Работал он не идеально, видимо поэтому начиная с версии `v2023.1.0` 
[его заменили на новое решение](https://roadrunner.dev/docs/releases-v2023-1-0/2023.x/en#the-reload-plugin-has-been-removed-from-the-default-plugins-list.-please-use-.pool.debug-true-instead).
**И если у вас по каким-то причинам Roadrunner версии <=** `v2.12.3`, то
необходимо будет использовать [Auto-Reloading](https://roadrunner.dev/docs/plugins-reload/2.x/en).
{:.info}

Настроим IDE PhpStorm. Зайдем в настройки `Settings` -> `PHP` -> `Servers` -> `Add`. Имя сервера зададим `rr`,
Host `localhost`, выберем галочку `Use path mapping`, установим маппинг текущей директории проекта на `/app`.

![PhpStorm настройки сервера Xdebug](/assets/articles/php-grpc-server/xdebug-server-settings-phpstorm.png)

В настройках дебага `Settings` -> `PHP` -> `Debug` сверим, что установлены порты `9000,9003`, внешние соединения разрешены.

![PhpStorm настройки, вкладка Debug Xdebug](/assets/articles/php-grpc-server/xdebug-debug-settings-phpstorm.png)

Я предпочитаю выставлять максимальное количество одновременных соединений в значение 1, но это вкусовщина. И это
только при условии, что мы дебажим одновременно только один сервис. Если же к примеру мы в этом же приложении
будем запускать код grpc клиента, который будет обращаться по сети к grpc серверу, то одного коннекта будет недостаточно,
и скорее всего отладка встанет в ожидании
{:.info}

После чего:

- Установим *breakpoint*, например, в недавно реализованном сервисе `CurrencyRateService`
- "Включим трубку" в PhpStorm (иконка телефонной трубки `Start Listening for PHP Debug Connections`)
- Запустим контейнер `docker-compose up -d`
- Повторим вызов метода с Postman - а 

и как результат, мы должны увидеть успешную точку останова:

![PhpStorm точка останова Xdebug](/assets/articles/php-grpc-server/xdebug-break-at-line-phpstorm.png)

Хоть это и так должно быть понятно, но не могу не отметить, что данный сетап в целом **не рассчитан на запуск в продакшене**. Все
вышеперечисленные настройки в таком виде **не должны ехать в прод**. Продакшн docker образ должен быть собран без `xdebug`,
а `pool.debug` не должен быть включен, т.к. это все очень негативно сказывается на производительности, и включается сугубо
для локальной разработки.
{:.warning}

## PHP клиент к gRPC серверу

По сути, при кодогенерации серверного кода нам был сгенерирован также и клиентский код. Находится он там же, в
директории `./generated`, конкретно класс клиента `GRPCContracts\Currency\CurrencyRateServiceClient`.

Мы хотим иметь консольную команду, которая будет инжектить в себя этот клиентский сервис, обращаться через него
к поднятому в той же docker сети gRPC серверу, получать данные и выводить их на экран. Для этого нам нужно:

- Добавить в окружение `php-cli` docker контейнер, из-под которого будем запускать консольную команду
- Написать саму консольную команду
  - Подключить в автолоад `composer` директории `./generated/Bootloader` и `./generated/Config`
  - Зарегистрировать gRPC клиент в di контейнере приложения, подключив уже любезно сгенерированный bootloader
  - Заинжектить в консольную команду сгенерированный gRPC клиент
- Добавить в `.env` переменную окружения `CURRENCY_RATE_SERVICE_HOST`

### Docker контейнер php-cli

Консольную утилиту запускать будем из отдельного `php-cli` контейнера,
который опишем в уже имеющемся `docker-compose.yml`:

```yml
# ./docker-compose.yml
...
  php-cli:
    build:
      dockerfile: ./deployment/docker/php-cli/Dockerfile
    user: '1000:1000'
    environment:
      PHP_IDE_CONFIG: 'serverName=rr'
    tty: true
    volumes:
      - ./:/app
      - ./deployment/docker/roadrunner/xdebug.ini:/usr/local/etc/php/conf.d/xdebug.ini
...
```

Как видим, он собирается из другого `Dockerfile`, в котором нет за ненадобностью утилит 
`protoc`, `git`, расширения `protobuf`. Зато есть extension `grpc`, нужный для запуска клиентского кода.

Настройки xdebug аналогичные.

```bash
# ./deployment/docker/php-cli/Dockerfile

# Базируемся на официальном образе php cli alpine
FROM php:8.2-cli-alpine

# Рабочая директория /app
WORKDIR /app

# Копируем скрипт install-php-extensions от mlocati
# Данный проект рекомендован на официальной странице образов php https://hub.docker.com/_/php
COPY --from=mlocati/php-extension-installer /usr/bin/install-php-extensions /usr/bin/

# Устанавливаем необходимые расширения
RUN install-php-extensions opcache zip sockets xdebug \
    grpc

# Копируем composer
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer
```

Специально разделил контейнеры клиента и сервера чтобы наглядно показать, что они содержат разный набор
расширений и утилит. Могут содержать и разную кодовую базу в теории, но чтобы не усложнять мануал, код у нас
содержится в рамках одного php приложения.
{:.info}

### Composer autoload

В автолоад composer - а, в секцию `autoload.psr-4` добавим строки:

```json
...
"autoload": {
        "psr-4": {
            ...
            "App\\Application\\Bootloader\\": "generated/Bootloader",
            "App\\Application\\Config\\": "generated/Config",
            ...
        }
    },
...
```

Тем самым подключим сгенерированные классы, необходимые для подключения клиентского сервиса к di контейнеру 
фреймворка.

Не забудем применить изменения в `composer`:

```bash
docker-compose run rr composer du
````

### Подключим di bootloader

Добавим в класс `App\Application\Kernel` в секцию `LOAD` наш бутлоадер:

```php
...
// Grpc service bootloader
\App\Application\Bootloader\ServiceBootloader::class,
...
```

### Пишем консольную команду

```php
<?php
// ./app/src/CurrencyRateCliCommand.php

declare(strict_types=1);

namespace App;

use GRPCContracts\Currency\CurrencyRateServiceInterface;
use GRPCContracts\Currency\GetRateRequest;
use Spiral\Console\Command;
use Spiral\RoadRunner\GRPC\Context;

final class CurrencyRateCliCommand extends Command
{
    // Сигнатура команды = имя команды + аргумент "ticker"
    public const SIGNATURE = 'grpc:get-currency-rate {ticker : Currency ticker}';
    public const DESCRIPTION = 'Get currency rate command';

    // Инжектим экземпляр клиента по интерфейсу
    public function __construct(
        private readonly CurrencyRateServiceInterface $currencyRateGrpcClient,
    ) {
        parent::__construct();
    }

    protected function perform(): int
    {
        // значение тикера берем из аргумента команды
        $ticker = $this->argument('ticker');

        // создаем запрос, помещаем в него тикер
        $request = new GetRateRequest();
        $request->setTicker($ticker);

        // вызываем gRPC метод клиента, получаем ответ от сервера
        $response = $this->currencyRateGrpcClient->GetRate(new Context([]), $request);

        // выводим результат на экран
        $this->output?->writeln("{$response->getTicker()}: {$response->getRate()}");

        return self::SUCCESS;
    }
}
```

### .env

В `.env` файл добавим в конец переменную окружения со значением `хост:порт` запущенного gRPC сервера, 
к которому будет обращаться клиент:

```bash
# grpc app
CURRENCY_RATE_SERVICE_HOST=rr:9001
```

`rr` - имя сервиса с roadrunner контейнером из `docker-compose.yaml`. Оба контейнера, `rr` и `php-cli`, будучи
запущенными в одной docker сети, могут обращаться друг к другу по алиасу сервиса, который docker любезно
разрезолвит в ip-адрес внутри этой сети, уже по факту.

### Запуск

Остановим и запустим с пересборкой наши сервисы:

```bash
docker-compose stop && docker-compose up -d --build
```

Из под контейнера `php-cli` запустим новую консольную команду, передав тикер `TRYUSD`
как аргумент:

```bash
docker-compose exec php-cli php app.php grpc:get-currency-rate TRYUSD
```

Вывод от команды:

```
TRYUSD: 75.00
```

Т.е. все работает и клиент и сервер. Запущенный сервер слушает порт 9001. 
Клиент к серверу обращается по сети через вызов gRPC клиента, получает
и выводит ответ на экран.

## Планы

В данном материале мы использовали самый быстрый путь к достижению результата, а именно использовали `Spiral Framework`.
Ничего против него не имею, мне он даже понравился. Но выбор/смена фреймворка является важнейшим шагом как на старте
проекта, так и в рамках существующих проектов. И многим `Spiral` может попросту не подойти по разным причинам.

Поэтому есть мысли по написанию подобных гайдов под `Symfony`, или даже без фреймворка. И если интересно
продолжение в этом ключе, то [ставь кита](https://t.me/igancev_ru/18) (слона там нет, а жаль)
как реакцию на соответствующий пост в телеграм канале.

## Заключение

В данном примере многое было сделано на скорую руку. Многие аспекты намеренно были мной упрощены и опущены.
Но не потому, что по другому не умею. А ради того, чтобы
как можно легче донести основной посыл. Показать пример пусть простейшего, но рабочего **gRPC сервера на php**.
Чутка с натяжкой, ведь всю работу за нас делает софт `Roadrunner`, написанный на `Go`. Но я в этом не вижу ничего плохого.
Серверный код логики нашего приложения мы пишем на `php`.
А Roadrunner действительно классный и гибкий продукт, советую изучить его глубже, в сети куча интересных материалов.

## Комментарии

Любые обсуждения по теме приветствуются под [постом данной статьи в
телеграм канале](https://t.me/igancev_ru/18). Буду рад, если поделитесь своим опытом в затронутом вопросе. 
Или найдете и укажете на неточности. 
