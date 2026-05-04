---
title: "Числа прописью на венгерском языке через Foma"
date: 2024-05-25
lastmod: 2026-05-04
description: "Небольшая заметка о том, как в языковом модуле RHVoice числа, записанные цифрами, превращаются в венгерские слова через Foma."
tags: ["программирование", "rhvoice", "foma", "эзотерика", "занятно"]
---

Давненько я ничего не писал в своём программерском блоге, но пройти мимо такого странного инфоповода просто не мог.

Дело в том, что, как ни странно, мы вместе со Звонимиром, лингвистом из Хорватии, разрабатываем венгерский языковой модуль для синтезатора RHVoice.

Да да, русский и хорват работают над венгерским языком. Чего только в этом мире не случается.

И сегодня я написал по сути небольшой, но важный код, преобразующий написанные цифрами числа в их буквенное представление.

Это как:

```text
1123 = одна тысяча сто двадцать три
````

Только для венгерского языка.

И именно этим кодом я хочу с вами поделиться, ибо код этот страшен и написан на таком странном эзотерическом языке под названием Foma.

## Код

Добавлю переводов, ибо читатель вероятно с венгерским языком не знаком.

```foma
define NZDigit 1|2|3|4|5|6|7|8|9;
define Digit %0|NZDigit;
define PlusOrMinus %+|%-;
define Number [(PlusOrMinus) [%0|[NZDigit Digit^<15]] (%%|%°)];
define RB (%%|%°) .#. ;

# Цифры от "один" до "девять" прописью
define Units %0:0|1:egy|2:kettő|3:három|4:négy|5:öt|6:hat|7:hét|8:nyolc|9:kilenc;

# Числа от "десять" до "девятнадцать" прописью
define Teens 1:0 [%0:tíz|1:tizenegy|2:tizenkettő|3:tizenhárom|4:tizennégy|5:tizenöt|6:tizenhat|7:tizenhét|8:tizennyolc|9:tizenkilenc];

# Круглые десятки прописью от "двадцать" до "девяносто"
define Tens %0:0|2:húsz|3:harminc|4:negyven|5:ötven|6:hatvan|7:hetven|8:nyolcvan|9:kilencven;

# Круглые сотни прописью
define Hundreds %0:0|1:egyszáz|2:kettőszáz|3:háromszáz|4:négyszáz|5:ötszáz|6:hatszáz|7:hétszáz|8:nyolcszáz|9:kilencszáz;

# Тут думаю всё понятно, но nulla - это ноль
define UpToThousand %0:nulla .P. [[(Hundreds) Teens] | [((Hundreds) Tens) Units]];

define DigitsToWords 
UpToThousand @-> ,, 
%+ @-> plusz ,, 
%- @-> minusz ,, 
%% @-> százalék ,, # название символа "процент"
%° -> fok ; # Название символа "градус"

define InsertThousands 
[..] -> ezer || Digit _ Digit^3 RB; # ezer - тысяча

define InsertMillions 
[..] -> millió || Digit _ Digit^3 ezer;

define InsertBillions 
[..] -> milliárd || Digit _ Digit^3 millió;

define InsertTrillions 
[..] -> billió || Digit _ Digit^3 milliárd;

define RemoveSkippedParts 
%0 %0 %0 [ezer|millió|milliárd|trillió] -> 0 ;

# Немного лингвистики, объяснение в комментарии не влезет
define FixTwenty 
húsz @-> huszon || _ egy|kettő|három|négy|öt|hat|hét|nyolc|kilenc;

# Ещё немного лингвистики, потому что "egyszáz" - это как на русском сказать "одинсто"
define FixHundred 
egyszáz -> száz || .#. _ ;

# Потому что венгры никогда не говорят "одна тысяча" - всегда только "тысяча"
define FixThousand 
egy -> 0 || .#.|millió|milliárd|billió _ ezer ;

# Финал, ради которого и страдали
regex 
Number .o. 
InsertThousands .o. 
InsertMillions .o. 
InsertBillions .o. 
InsertTrillions .o. 
RemoveSkippedParts .o. 
DigitsToWords .o. 
FixTwenty .o. 
FixHundred .o. 
FixThousand;
```

## Что это вообще такое

Foma - это как регулярные выражения, только на максималках.

Помимо обычных регулярок тут есть:

* правила трансформации по определённым условиям;
* встроенные замены;
* композиция правил;
* конечные автоматы;
* ещё куча адских приколов, разобраться с которыми получается далеко не сразу.

Но именно вот такие страшные конструкции лежат в основе любого языка в RHVoice.

И работает это пусть и сложно, но офигеть как быстро.

## Что делает этот конкретный кусок

Если сильно упростить, тут происходит несколько этапов.

Сначала описывается, что вообще считается числом:

```foma
define NZDigit 1|2|3|4|5|6|7|8|9;
define Digit %0|NZDigit;
define PlusOrMinus %+|%-;
define Number [(PlusOrMinus) [%0|[NZDigit Digit^<15]] (%%|%°)];
```

То есть число может быть:

* положительным;
* отрицательным;
* с процентом;
* с градусом;
* с ограничением по длине.

Потом задаются базовые слова для единиц, десятков и сотен:

```foma
define Units %0:0|1:egy|2:kettő|3:három|4:négy|5:öt|6:hat|7:hét|8:nyolc|9:kilenc;
```

```foma
define Tens %0:0|2:húsz|3:harminc|4:negyven|5:ötven|6:hatvan|7:hetven|8:nyolcvan|9:kilencven;
```

```foma
define Hundreds %0:0|1:egyszáz|2:kettőszáz|3:háromszáz|4:négyszáz|5:ötszáz|6:hatszáz|7:hétszáz|8:nyolcszáz|9:kilencszáz;
```

Дальше начинается вставка тысяч, миллионов, миллиардов и прочих радостей:

```foma
define InsertThousands 
[..] -> ezer || Digit _ Digit^3 RB;
```

```foma
define InsertMillions 
[..] -> millió || Digit _ Digit^3 ezer;
```

И в конце вся эта красота собирается в один конвейер:

```foma
regex 
Number .o. 
InsertThousands .o. 
InsertMillions .o. 
InsertBillions .o. 
InsertTrillions .o. 
RemoveSkippedParts .o. 
DigitsToWords .o. 
FixTwenty .o. 
FixHundred .o. 
FixThousand;
```

Выглядит страшно. Работает быстро. Добро пожаловать в мир правил для синтеза речи.

## Маленькие правки после основной конвертации

Отдельно тут есть фиксы для случаев, где простая механическая сборка слова даёт не совсем правильную форму.

Например:

```foma
define FixTwenty 
húsz @-> huszon || _ egy|kettő|három|négy|öt|hat|hét|nyolc|kilenc;
```

То есть `húsz` (двадцать) превращается в `huszon`, если это, например, 21, 22, 23, а не просто 20.

Ещё пример:

```foma
define FixHundred 
egyszáz -> száz || .#. _ ;
```

Потому что не всё, что механически собирается из кусочков, должно так же механически звучать в реальной речи.

Языки, как обычно, решили, что программистам слишком легко живётся.

## Итог

Это всего лишь кусок кода для преобразования чисел в слова.

Но этот кусок находится на пересечении:

* программирования;
* лингвистики;
* синтеза речи;
* конечных автоматов;
* и какой-то локальной эзотерики.

Русский и хорват пишут венгерский модуль для RHVoice на языке Foma.

Нормальный вторник.
