---
title: "Nextcloud на Debian 12: Imagick установлен, а предупреждение не исчезает"
date: 2024-10-22
lastmod: 2026-05-04
description: "Короткое заклинание для Nextcloud на Debian 12, если php-imagick установлен, но проверка всё равно ругается на Imagick."
tags: ["linux", "debian", "nextcloud", "php", "imagick", "администрирование", "заклинание"]
---

Ставим Nextcloud на Debian 12, устанавливаем `php8.3-imagick`, а в настройках всё равно видим сообщение о том, что модуль Imagick не установлен или работает не полностью.

Решается неочевидно: нужно установить ещё один пакет:

```bash
sudo apt install libmagickcore-6.q16-6-extra
````

После этого обновляем страницу проверки в настройках Nextcloud, тем самым перезапуская все тесты работоспособности, и видим, что ошибка исчезла.

## Почему так

Судя по похожим случаям, проблема обычно не в том, что PHP-модуль Imagick вообще отсутствует, а в том, что установленной связке ImageMagick/Imagick не хватает поддержки некоторых форматов. В Nextcloud это часто всплывает как предупреждение про Imagick или SVG support. Пакет `libmagickcore-6.q16-6-extra` как раз добавляет дополнительные компоненты ImageMagick, из-за отсутствия которых такие проверки могут ругаться.

## Проверка

После установки можно проверить, что пакет стоит:

```bash
dpkg -l | grep libmagickcore-6.q16-6-extra
```

И что PHP видит Imagick:

```bash
php -m | grep -i imagick
```

Если используется PHP-FPM, на всякий случай можно перезапустить его:

```bash
sudo systemctl restart php8.3-fpm
```

Если Apache с mod_php:

```bash
sudo systemctl restart apache2
```

## Актуальность

Проверено на такой связке:

```text
Nextcloud 29
Debian 12
PHP 8.3
```

Для более новых версий Debian пакет может называться иначе. Например, в обсуждении Docker-образа Nextcloud для более свежей базы уже всплывал переход от `libmagickcore-6.q16-6-extra` к `libmagickcore-7.q16-10-extra`. Поэтому для Debian 12 команда выше актуальна, а для Debian 13 и новее название лучше проверить через `apt search`.

## Итог

Если после установки `php8.3-imagick` Nextcloud всё ещё ругается на Imagick, попробуйте:

```bash
sudo apt install libmagickcore-6.q16-6-extra
```

Не великое шаманство, но достаточно неочевидное, чтобы записать.
