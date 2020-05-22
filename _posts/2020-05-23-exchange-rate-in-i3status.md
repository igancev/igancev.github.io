---
title: Вывод курса валюты в i3status
excerpt: Напишем bash скрипт для получения курса валюты доллара и евро по отношению к рублю, и выведем их в i3status
keywords: i3status exchange rate курс валют i3wm i3bar arch linux
key: i3status
tags: i3status Bash i3wm Linux
slug: exchange-rate-in-i3status
---

Напишем bash скрипт для получения курса валюты доллара и евро по отношению к
рублю, и выведем их в `i3status`

<!--more-->

В предыдущей статье мы [настроили i3status](/2020-05-11-configuring-i3status-in-i3wm)
и научились добавлять к нему свои собственные блоки. Теперь добавим к строке
статуса текущие активные курсы интересующих нас валют.

Получать свежий курс валют будем с Центробанка по адресу
`https://www.cbr-xml-daily.ru/daily_json.js` при помощи `curl` в формате json.

`i3status` не очень комфортно настраивать для показа результата каких - то своих
команд или работы своих скриптов. Приходится оборачивать его вызов в свой bash
скрипт, и модифицировать json. Скрипт в итоге будет вызываться с периодичностью,
заданной в конфигурационном файле. У меня это 1 секунда, и, конечно же, нет
никакой необходимости обновлять курсы валют так часто. Поэтому наш скрипт
будет сохранять результаты полученных курсов валют в файлы. Поставим скрипт
на выполнение по расписанию с помощью `cron` с нужным интервалом,
а затем уже из файла мы можем выводить курсы валют с помощью штантого модуля
i3status **File Contents** (read_file).

Начнем с написания bash скрипта:

~/.config/i3status/currency.sh
```bash
#!/bin/bash

# контекст, директория хранения данных о курсах валют
contextPath="$HOME/.config/i3status/currency"

# файл json со всеми валютами, получаемыми из cbr
currencyJsonPath="${contextPath}/currency-cbr.json"

# обновление файла с json валют
function updateCurrencyCbr {
    curl https://www.cbr-xml-daily.ru/daily_json.js > $currencyJsonPath
}

# получение значения переданной валюты
# из JSON файла cbr
# округленного до 2-х знаков после запятой
function getRoundedCurrency {
	local currencyKey="$1"

	value=`cat ${currencyJsonPath} | jq .Valute.$currencyKey.Value`
	roundValue=`echo "scale=2;($value)/1" | bc`

	echo $roundValue
}

# получение стрелки вниз (падение валюты)
# или вверх (рост валюты)
function getCurrencyArrow {
	local currencyKey="$1"

	value=`cat ${currencyJsonPath} | jq .Valute.$currencyKey.Value`
	valuePrevious=`cat ${currencyJsonPath} | jq .Valute.$currencyKey.Previous`

	# если курс не поменялся
	if [[ `echo "$value==$valuePrevious" | bc` -eq 1 ]]; then
		echo ""
	# курс изменился в большую сторону
	elif [[ `echo "$value>$valuePrevious" | bc` -eq 1  ]]; then
		echo "↑"
	# курс изменился в меньшую сторону
	else
		echo "↓"
	fi
}

# сохранение значения переданной валюты
# и индикатора динамики изменения (стрелки) в отдельный файл
function saveCurrency {
	local currencyKey="$1"

	echo `getRoundedCurrency "$currencyKey"``getCurrencyArrow "$currencyKey"` > "$contextPath/$currencyKey"
}

### start ###

# создание директории, если не создана
mkdir -p "${contextPath}"

# обновление валют cbr
updateCurrencyCbr

# сохранение интересующих валют
saveCurrency "USD"
saveCurrency "EUR"

```

Исполним его:

```bash
~/.config/i3status/currency/currency.sh
```

В директории `~/.config/i3status/currency` появились файлы с подготовленными
для вывода файлами:

```bash
ls ~/.config/i3status/currency
```

```
currency-cbr.json  currency.sh  EUR  USD
```

Нас интересуют файлы с сегодняшним курсом `EUR` и `USD`:

```bash
cat ~/.config/i3status/currency/EUR
78.44↑

cat ~/.config/i3status/currency/USD
71.88↑
```

С содержимым все в порядке, теперь поставим скрипт на выполнение по расписанию
в `cron`. Курс на завтрашний день в ЦБ обновляется в 11:30, поэтому поставим
исполнение скрипта чуть позже на всякий случай, в 11:35, раз в сутки. Затем еще
раз, на всякий случай, через час, в 12:35.

```bash
crontab -e
```

```
35 11,12 * * * ~/.config/i3status/currency/currency.sh
```

По желанию можно поставить скрипт на выполнение при старте системы.
В `i3wm` я это сделаю в конфигурационном файле i3 `~/.config/i3/config`

```bash
exec --no-startup-id ~/.config/i3status/currency/currency.sh
```

Теперь при старте системы будет получен в фоне актуальный курс, а в 11.35 будет
обновлен. Осталось только вывести значение в `i3status`. Отредактируем
конфигурационный файл `~/.config/i3status/config`, добавив туда блоки
File Contents (read_file), по одному на каждую валюту:

```bash
...
order += "read_file $"
order += "read_file €"
...

read_file $ {
	path = "/home/igancev/.config/i3status/currency/USD"
	format = "%title: %content"
}

read_file € {
	path = "/home/igancev/.config/i3status/currency/EUR"
	format = "%title: %content"
}
```

![Вывод курсов валют в i3status](/assets/articles/currency-i3status/i3status-currency.png)

Вот и все. По аналогии можно добавить другие валюты, подсмотрев их код
в сохраняемом `currency-cbr.json`.

**Итог:** при желании и некотором навыке в `i3status` можно вывести много
полезной информации. Однако писать подобные скрипты, заботиться о своевременном
обновлении файлов, а также задействовать автозагрузку и `cron` кажется не лучшим
решением. Возможно, если ваши потребности далеко выходят за рамки `i3status`,
стоит задуматься об альтернативе. Например в `i3blocks` можно задавать
команды каждому блоку напрямую, а также задавать из конфигурационного файла
период обновления. Но это уже тема для другой статьи.
