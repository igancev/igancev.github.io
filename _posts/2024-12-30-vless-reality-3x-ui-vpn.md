---
title: Настройка надежного VPN сервера на протоколе VLESS + Reality
excerpt: Настроим собственный, устойчивый и надежный VPN сервер на основе протокола VLESS и шифрования Reality. Подключим к нему клиентские девайсы, а управлять им будем из красивого UI
keywords: Vless, Reality, TLS, Xray, 3x-ui, vpn, Простейший способ создания своего надежного VPN сервера на протоколе VLESS  Reality
key: vless-reality-3x-ui-vpn
tags: vpn vless 3x-ui linux vps
slug: vless-reality-3x-ui-vpn
mainImage: /assets/articles/vless/vless-reality-main.webp
---

![vless reality image](/assets/articles/vless/vless-reality-main.webp)

[👉👉Сразу перейти к установке 👈👈](#установка-3x-ui)

🤐 Материал носит **сугубо технический характер** и лишь описывает технологии
создания защищенных каналов связи, что необходимо **для обеспечения безопасности
приватных корпоративных сетей**. Ничего противозаконного.
{:.warning}

## Вступление

Материал ориентирован на рядовых пользователей и не требует глубоких знаний и умений в
computer science. Цель - в короткое время и **за минимум усилий**,
условно за одну введенную команду получить желанный, **рабочий результат**.

Ввиду упрощения настройки и подачи материала, тут не будет деталей и технических
подробностей, описывающих используемые протоколы и выбранные решения.

Поехали! 🚀

🔔 Подписывайтесь на [телеграм канал](https://t.me/igancev_ru), чтобы не пропустить
выход новых статей в блоге 🕺
{:.info}

## 3x-ui панель

[3x-ui](https://github.com/MHSanaei/3x-ui) - веб панель,
с помощью которой мы и будем управлять
нашим сервером. В ней мы создадим подключение, можно несколько и будем
добавлять клиентов к этим подключениям. Великолепнейшая вещь, существенно
упрощающая нам работу 👍

![3x-ui panel](/assets/articles/vless/3x-ui.png)

## Подготовка

### Арендуем vps сервер

- [Выбираем хостинг провайдера, у которого арендуем сервер](https://hip.hosting/?code=1e0e0f33769e9d6ed8f0).
  - Определяемся с желаемой страной размещения.
    - Одним из важнейших критериев тут является географическая близость. Мой выбор - Европа, Нидерланды.
  - Определяемся с системными требованиями. Достаточно будет
    *1cpu, 512Mb-1Gb RAM, KVM, 100-1000 Mbit/sec*.

Арендовать **недорогой и быстрый VPS для VPN** в Европе [можно тут](https://hip.hosting/?code=1e0e0f33769e9d6ed8f0) 👈🏾.
{:.success}

  - ОС сервера выбираем **Ubuntu 22.04/24.04**
  - Регистрируемся, оплачиваем
    - Кто с этим не сталкивался, там нет ничего необычного. Обычная регистрация как на любом сайте. В личном
      кабинете интересующую конфигурацию сервера добавляем в корзину, оплачиваем любым доступным из предложенных способов.
      *Реквизиты подключения к серверу по **ssh*** обычно приходят на электронную почту после его оплаты

### Подключаемся по ssh

Ранее я уже описывал этот процесс, можно почитать [тут](/2022-03-13-simple-and-fast-install-vpn-wireguard-docker#2-подключаемся-по-ssh).

## Установка 3x-ui

❗️ Внимание! Данная установка должна проводиться из под пользователя `root`.
{:.warning}

Нам потребуется запустить **всего один** [скрипт](https://github.com/igancev/install-3x-ui/blob/master/install-3x-ui.sh):

```shell
bash <(curl -Ls https://raw.githubusercontent.com/igancev/install-3x-ui/master/install-3x-ui.sh)
```

В процессе установки скрипт спросит лишь один вопрос, _не желаем ли мы установить
особый порт для панели?_

```
Would you like to customize the Panel Port settings?
(If not, a random port will be applied) [y/n]: n
```

Отклоняем предложение вводом `n`, нажимаем `Enter` и дожидаемся окончания
установки, пока не увидим следующий вывод:

```shell
#######
Username: XJZrXVRYMj
Password: kMJ5efXfYd
Access URL: https://88.145.205.44:13974/xaa0swkOI1nqgae/
#######
```

Перед нами ссылка доступа в 3x-ui панель, логин и пароль.

При первом запуске браузер ругнется на неприватное соединение. Предупреждение можно смело игнорировать.
С соединением все в порядке, оно защищено самоподписанным сертификатом, выпущенным
при установке. Подтвердим единожды переход в панель:

![proceed-unsafe-browser-connection.png](/assets/articles/vless/proceed-unsafe-browser-connection.png)

Залогинимся для дальнейшей несложной настройки:

![3x-ui login page](/assets/articles/vless/3x-ui-intro.png)

## Настройка подключения VLESS + Reality

Нам необходимо единожды настроить подключение и добавить нужное количество
клиентов. В дальнейшем в панель мы будем заходить только для управления
клиентами.

Перейдем в раздел `Inbounds/Подключения` и нажмем кнопку `Add Inbound/Добавить подключение`.

1. Пункт `Security/Безопасность` выставляем в значение `Reality`
2. Нажимаем кнопку `Get New Cert`
3. Разворачиваем блок `Client/Клиент`, выставляем пункт `Flow` в значение`xtls-rprx-vision`
4. _(Опционально)_ В поле `Remark/Примечание` пишем желаемое нами наименование подключения,
   например: `My awesome inbound`
5. _(Опционально)_ В поле `Client -> Email` пишем любое желаемое нами имя
  клиента, совсем не обязательно email, например: `My awesome client`

![3x-ui add inbound](/assets/articles/vless/add-inbound.png)

В списке подключений теперь наблюдаем наше подключение и созданного клиента.
Рекомендуется под каждое клиентское устройство создавать отдельного клиента,
что можно сделать, выбрав соответствующий пункт меню из списка.

![3x-ui inbound list](/assets/articles/vless/inbounds-list.png)

На этом настройка сервера завершена, можно подключать клиентские девайсы 🕺

## Настройка клиентских девайсов


На устройство необходимо установить программу, которой будем подключаться
к настроенному серверу. Для каждой ОС существуют свой набор клиентского ПО,
на выбор. Всех их объединяет одно, это способ подключения. Их всего два:

### Qr-код

Этот способ наиболее удобен при подключении мобильных устройств, изх клиентского
ПО которых можно добавить подключение путем сканирования QR кода камерой.

![3x-ui qr code](/assets/articles/vless/3x-ui-qr.png)

В UI панели, списке клиентов, напротив каждого клиента, есть иконка
с `QR` кодом. Нажимая на нее на экране появится `QR`, ну дальше вы поняли.

### Импорт url строки подключения

Импорт через `URL` более универсален. URL можно отобразить и скопировать, нажав
на иконку с информацией, напротив каждого клиента:

![3x-ui url copy](/assets/articles/vless/url-copy.png)

Данную строку с `URL` можно импортировать в любой клиент, на любой операционной
системе, или отправить другу.

### Клиенты под девайсы

- Android
  - [NekoBoxForAndroid](https://github.com/MatsuriDayo/NekoBoxForAndroid) - свежую apk искать на [странице с релизами](https://github.com/MatsuriDayo/NekoBoxForAndroid/releases)
  - [v2rayNG](https://play.google.com/store/apps/details?id=com.v2ray.ang&pli=1)
  - [v2RayTun](https://play.google.com/store/apps/details?id=com.v2raytun.android&hl=ru)
- Ios
  - [V2Box](https://apps.apple.com/ru/app/v2box-v2ray-client/id6446814690?platform=iphone)
  - [Streisand](https://apps.apple.com/ru/app/streisand/id6450534064?platform=iphone)
  - [FoXray](https://apps.apple.com/ru/app/foxray/id6448898396?platform=iphone)
  - [Shadowrocket](https://apps.apple.com/ru/app/shadowrocket/id932747118?platform=iphone)
  - [v2RayTun](https://apps.apple.com/ru/app/v2raytun/id6476628951?platform=iphone)
- Linux
  - [Nekoray](https://github.com/MatsuriDayo/nekoray/) - [релизы](https://github.com/MatsuriDayo/nekoray/releases)
  - [sing-box](https://github.com/SagerNet/sing-box)
- MacOS
  - [V2Box](https://apps.apple.com/ru/app/v2box-v2ray-client/id6446814690)
  - [FoXray](https://apps.apple.com/ru/app/foxray/id6448898396)
  - [Streisand](https://apps.apple.com/ru/app/streisand/id6450534064)
  - [V2RayXS](https://github.com/tzmax/V2RayXS)
  - [NekoRay/NekoBox for macOS](https://github.com/abbasnaqdi/nekoray-macos)
  - [Furious](https://github.com/LorenEteval/Furious/)
- Windows
  - [Furious](https://github.com/LorenEteval/Furious)
  - [InvisibleMan-XRayClient](https://github.com/InvisibleManVPN/InvisibleMan-XRayClient)
  - [Nekoray](https://github.com/MatsuriDayo/nekoray/)

## Telegram канал

Обсудить тему можно [под постом в телеграм канале](https://t.me/igancev_ru/23),
на который, к слову, **можно подписаться**, чтобы не пропустить контент 🔔


