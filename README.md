* [Ликбез](#Ликбез)
  * [Загрузка](#Загрузка)
    * [BIOS](#bios)
    * [Типы разметки ЗУ](#Типы-разметки-ЗУ)
      * [MBR](#mbr)
        * [Разделы](#Разделы)
      * [GPT](#gpt)
      * [Конвертация](#Конвертация)
    * [Boot loader (загрузчик)](#boot-loader-загрузчик)
      * [MBR](#mbr-1)
      * [GPT](#gpt-1)
  * [Некоторые вопросы, на которые нужно знать ответы](#Некоторые-вопросы-на-которые-нужно-знать-ответы)
    * [Настройки UEFI](#Настройки-uefi)
      * [AHCI (и RAID) на Win7](#ahci-и-raid-на-win7)
      * [AHCI на Win8+](#ahci-на-win8)
    * [Создание загрузочной флешки](#Создание-загрузочной-флешки)
    * [Работа с разделами](#Работа-с-разделами)
    * [Запуск программ из *boot.wim*](#Запуск-программ-из-bootwim)
* [Практика](#Практика)
  * [Стандартная установка](#Стандартная-установка)
    * [Ошибки-ошибочки](#Ошибки-ошибочки)
  * [Распаковка при помощи WinNTSetup3](#Распаковка-при-помощи-winntsetup3)
  * [Создание своего образа для быстрого развёртывания](#Создание-своего-образа-для-быстрого-развёртывания)
  * [Развёртывание из образа install.wim средствами wimlib-imagex](#Развёртывание-из-образа-installwim-средствами-wimlib-imagex)

Установка ОС шиндовс — дело нехитрое. Но при попытке сотворить что-то нестандартное, анончик обычно начинает сосать хуй из-за того, что кнопочки — это не всё, что присутствует в стандартном комплекте поставки. Да и кнопочки нужно жать понимаючи и последствия осознавая.

> () — Круглые скобки используются для вставки специфической и необязательной информации
> [] — Квадратные скобки обозначают второй уровень вложенности скобок (потому что мне так удобно [нет, правда])

# Ликбез

## Загрузка

### BIOS

В самом самом начале загружается BIOS, который может быть представлен в двух \(на самом деле нет\) вариациях: BIOS и UEFI \(по сути тот же BIOS, но продвинутый\). Эта сущность отвечает за:

1. Тестирование компонентов при загрузке \(POST\). Компьютер пищит из-за аппаратных ошибок именно на этой стадии.
2. Первоначальную инициализацию устройств \(ибо содержит некоторый набор "драйверов" — firmware)
   >При загрузке в безопасном режиме ОС шиндошс использует в основном драйвера из биоса
3. Передачу управления загрузчику \(исполнимому файлу. Линекс, например, может загружаться напрямую, без загрузчика [efistub]\)

|                                 | BIOS                             | UEFI                                     |
| :------------------------------ | :------------------------------- | :--------------------------------------- |
| Серийный запуск                 | 1981                             | ~2012. Если мать выпущена позднее, с вероятностью 95% на ней УЕФИ |
| Поддерживаемые типы разметки ЗУ | MBR                              | MBR, GPT                                 |
| Разрядность кода                | 16 \(особого значения не имеет\) | 32, 64                                   |

> 32-х разрядные уефи производились только на первых порах, где-то до 10–11 годов. В большинстве (95%) случаев битов — ровно 64

### Типы разметки ЗУ

После загрузки BIOS пытается передать управление первому загрузчику, который будет обнаружен на ЗУ. То, что будет искать этот самый загрузчик, и как он будет это искать, зависит от типа разметки ЗУ. Поэтому для начала проясним ситуацию с ними:

Как MBR, так и GPT в своей сути являются просто **списком разделов** на данном ЗУ. А разница только в реализации \(ограничениях и плюшках\).

|                         | MBR                                      | GPT                                      |
| :---------------------- | :--------------------------------------- | :--------------------------------------- |
| Серийный запуск         | 1983                                     | 2010                                     |
| Количество разделов     | Изначально — [не более 4][MBR#Sector_layout] | 128 шиндовс, 255 линукс                  |
| Отказоустойчивость      | Информация о разделах хранится в одном месте | Информация о разделах хранится и в начале и в конце |
| Адресация секторов      | 32-хбитная                               | 64-хбитная                               |
| Адресуемое пространство | 2 ТиБ ≈ 2,2 ТБ                           | 8 ТиБ ≈ 8,8 ТБ                           |

Надо отметить, что тип разметки никак не привязан к накопителю: GPT можно запилить хоть на Seagate Barracuda 80Gb 15-летней давности. Это всего лишь кучка битов, расположенных в нужных местах, поэтому MBR можно преобразовать в GPT абсолютно элементарно.

####MBR

MBR физически располагается в первом секторе (первые 512 байт) ЗУ и содержит следующие элементы:

| Структура               | Размер (байт) |
| ----------------------- | ------------- |
| Исполнимый код          | 446           |
| Информация о разделе №1 | 16            |
| Информация о разделе №2 | 16            |
| Информация о разделе №3 | 16            |
| Информация о разделе №4 | 16            |
| Сигнатура               | 2             |

**Исполнимый код**: стандартный костыль, передающий загрузку дальше по цепочке.

**Информация о разделе** состоит из:

1. Статуса (загрузочный [80h] или нет [00h])
2. Расположения на диске
3. Типа (07h — NTFS, 0Ch — FAT32, 83h — Linux, etc.)

**Сигнатура**: (55AAh) определяет, что сектор содержит именно MBR, а не что-то иное.

##### Разделы

Разделы идут не сразу после MBR, а с некоторым *смещением* (64 сектора, 32КБ), которое хитрые людишки решили приспособить под продолжение исполнимого кода, лежащего в самой MBR.

Зачастую (то есть практически всегда), в начале раздела располагается **VBR** (**PBR**) (Volume/Partition Boot Record). Практически копия MBR, даже с загрузочным кодом. Может быть интересна лишь в случае записи ФС напрямую на ЗУ. Например, так:  ```$ mkfs.msdos /dev/sdb```

Есть также и костыль, позволяющий городить дополнительные разделы — так называемая EBR (Extended Boot Record). Также являющая собой кастрированный (функционально, не по размеру) MBR. Как и MBR, может содержать в себе загрузочный код, но для этой цели пользуется только крайне отбитыми личностями, поэтому для нас не очень интересна.

#### GPT

Структура — примитивнее некуда:

![https://en.wikipedia.org/wiki/GUID_Partition_Table][guid pic]
>LBA адресует сектора (участки по 512 байт)

GPT имеет обратную совместимость с MBR. Так что при желании можно впердолить на GPT загрузчик для MBR \(вот только нужно ли?\)

#### Конвертация

|                   | MBR > GPT | GPT > MBR |
| ----------------- | --------- | --------- |
| С потерей данных  | diskpart* | diskpart* |
| Без потери данных | [gptgen]  | [Paragon] |

- Работа с diskpart описана [ниже](#Работа-с-разделами)

### Boot loader \(загрузчик\)

В MBR также называется VBR \(PBR\). В EFI-системах — boot loader.

Эта штука предназначена для поиска самого ядра на разделе и выбора определённой загрузочной записи.

Поиск загрузчика происходит по следующим схемам:

#### MBR

1. В соответствии с настройками ("First boot device...", "Second...") BIOS выбирает устройство, с которого будет грузиться (жесткий диск, флешка, либо дисковод)
2. С устройства читается первый сектор, в котором располагается MBR
3. Выбирается раздел для загрузки (**активный раздел** [тот, который 80h])
4. Выполняется исполнимый код из начала **MBR**
5. Код из MBR передаёт управление коду из **VBR** на активном разделе
6. VBR уже умеет читать файловую систему раздела и без труда находит и передаёт управление файлу, на котором он завязан. Это уже и есть тот самый boot loader

Вкратце, **MBR > VBR > Boot loader**

Самые распространенные загрузчики:

|                |   NTLDR    |                 BOOTMGR                 | [grub4dos] |
| :------------- | :--------: | :-------------------------------------: | :--------: |
| Возможности    | Windows XP |                Windows7+                |    ISO     |
| Файл           |   ntldr    |                 bootmgr                 |   grldr    |
| Настройки      |  boot.ini  |                   BCD                   |  menu.lst  |
| Редактирование |  Блокнот   | bcdedit, [BootIce], [Visual BCD Editor] |  Блокнот   |

**NTLDR**

Старый загрузчик. Может загружать шиндошс XP и разные линексы.

**BOOTMGR**

Загрузчик поновее, посложнее. Записи уже блокнотом не подредактируешь, нужен спец софт.

Может загружать как XP, так и все последующие шиндовсы. И линексы тоже.

Стандартное расположение файла конфигурации загрузки:

|        | MBR             | GPT                       |
| :----- | :-------------- | ------------------------- |
| Путь   | */Boot/BCD*     | */EFI/Boot/Microsoft/BCD* |
| Раздел | Активный раздел | *ESP*                     |

**grub4dos**

Появился как ответвление от проекта GRUB — умолчательного загрузчика в линексе.

Могёт в загрузку ISO (важно, чтобы файл был дефрагментирован), а также линексов.

------

> **Каждый загрузчик может передавать управление любому другому. Так из grub4dos можно загрузить BOOTMGR, а из NTLDR ядро линекса.**

------

Для того, чтобы загрузить конкретный boot loader, очевидно, необходимо построить правильную цепочку из MBR и VBR. А так же обеспечить наличие файла загрузчика и файла конфигурации на активном разделе.

------

Некоторые маргинальные загрузчики для MBR: [PLoP], [XorBoot], [Smart BootManager], [GAG], [GRUB2WIN]

#### GPT

С этой штукой всё и проще и сложнее. Есть два варианта:

**a**. GPT работает под BIOS: в дело вступает слой совместимости, и процесс загрузки идёт по типу MBR, только в качестве активного раздела выбирается ESP.

**b**. GPT установлена на систему с UEFI.

Тут надо немного рассказать о возможностях самого UEFI: UEFI — сам себе загрузчик.

1. За счёт того, что искоробки он может работать и с FAT, и с FAT32 \(можно также поставить [драйверы][efi drivers] для NTFS или EXT, например\)
2. За счёт того, что опять же искоробки он предоставляет возможность создавать загрузочные записи и схоронять их в том же UEFI в энергонезависимой памяти.

Процесс загрузки на UEFI происходит так:

1. Происходит попытка загрузить файл из первой по приоритету записи UEFI.
2. С раздела, тип которого установлен в ESP \(EFI System Partition\) загружается файл /EFI/boot/bootx64.efi

**!!! Уточнить**

**!!! Дополнить описанием ESP**

Загрузчики EFI: systemd-boot, rEFInd, GRUB

Утилиты для редактирования записей UEFI: [BootIce], [EasyUEFI], efibootmgr (Linux), [UEFI Shell]

## Некоторые вопросы, на которые нужно знать ответы

### Настройки UEFI

**```Алярм! Только Win7! Win8+ таких проблем не имеют```**

UEFI обычно присутствует там, где уже впилены и AHCI, и USB 3.0 то есть то, во что стандартный *boot.wim* не могёт. У анончика появляется выбор:

1. Пердолить *boot.wim* для добавления поддержки недостающих функций

   (как вариант просто взять *boot.wim* из установочного образа Win8+)

2. Либо же просто выключить данные фичи на время установки

Думаю, очевидно, какой вариант выберем мы.

```
AHCI       → SATA или же Compatible
USB Legacy → ON
CSM        → ON
```

#### AHCI (и RAID) на Win7

*Зачем?* [NCQ]

*Как?* Элементарно:

1. Устанавливается [этот][AHCI RAID enable] твик реестра (ни разу не мокрописька. [Пруф][AHCI MS solution])

2. Собственно *Compatible* переключается на *AHCI*

3. ?????

4. PROFIT!

#### AHCI на Win8+

Достаточно переключиться на *AHCI* и перезагрузиться в безопасном режиме.

### Создание загрузочной флешки

Тут мудствовать лукаво не над чем: есть замечательная вещь с одной кнопкой — [Rufus]. Поэтому тут речь пойдёт именно о подводных:

1. Эта вещь сливает некоторые данные при проверке обновления. Отключается в "О программе..." → "Обновления"
2. На WinXP можно запиливать флешки для UEFI, если закинуть в папку с руфусом *wimgapi.dll* (его можно достать из *WinNTSetup3\Tools\x86*)

### Работа с разделами

Можно, конечно, воспользоваться жирненькими пакетами от [Acronis], [MiniTool] или ещё каким [парагоном][Paragon], но следует знать, что в ОС шиндовс есть довольно мощная консольная утилита **diskpart.exe**

Необходимый минимум:

```
list   <disk|partition>     — показать подключенные ЗУ|разделы на выбранном диске
select <disk|partition> <X> — выбрать ЗУ|раздел с номером X

clean             — очистить таблицу разделов ЗУ
convert [gpt|mbr] — преобразовать ЗУ в GPT или MBR

create partition primary [size=X]  — создать на выбранном ЗУ базовый раздел [размером X Мб]
format fs=<ntfs|fat32|fat> [quick] — форматировать выбранный раздел в ntfs, fat32 или fat [не затирая нулями информацию, находившуюся ранее]

assign [letter=X] — назначить выбранному разделу букву [букву X]
remove [letter=X] — удалить букву выбранного раздела [или просто удалить букву X]
```

### Запуск программ из *boot.wim*

На каком-нибудь работающем компухтере создаём папку на загрузочной флешке (например, *Soft*)

1. ```Shift+F10``` — запускаем командную строку

2. ```cd /d X:``` — судорожно ищем нашу флешку, подставляя буквы на место *X*.

   > Проверить, что это именно искомая флешка можно командой ```dir```.
   > В списке будет видна папка *Soft*

3. ```cd Soft``` — переходим в папку *Soft*

4. ```wimlib-imagex.exe``` — запускаем то, что на нужно

   > В командной строке есть автодополнение: достаточно ввести часть имени файла ```wimli``` и нажать **Tab**, как содержимое будет дополнено до ```wimlib-imagex.exe```

# Практика

## Стандартная установка

Тутошки рассматривается самый что ни на есть стандартный способ установить ОС Шиндошс 7. С некоторыми прояснениями.

1. Загрузка минимального Шиндовс (WinPE) из *sources\boot.wim*. Содержимое образа распаковывается в создаваемый RAM-диск (диск в оперативной памяти, скорость чтения с которого катастрофически выше, чем с HDD)
   ![][standard pic 1]

2. Запускается
   ![][standard pic 2]

3. Поскольку данный гуидо именно про установку, восстановление рассмотрено не будет. По крайней мере, пока.

   Обновление также рассматриваться не будет, ибо зачастую проще установить с нуля.

4. По нажатию на "Настройку диска" установщик ОС шиндовс предоставляет кое-какие возможности для разметки, но пользоваться этим довольно опасно — **изменения применяются моментально**.

   На EFI ОС шиндовс требует небольшого раздела для создания ESP
   ![][standard pic 3]
   ![][standard pic 4]

Дальнейшие пункты разбирать смысла не вижу, ибо заполнить поля и кликнуть оставшиеся "Далее" сможет и распоследняя обезьяна. Гораздо интереснее остановиться на тез местах, в которых установщик может внезапно начать копротивляться.

### Ошибки-ошибочки

Они логично проистекают из ограничений оригинального установщика:

1. Доступны только схемы BIOS+MBR и UEFI+GPT
2. Нельзя отказаться от установки загрузчика

Посему для запиливания уникальной™ конфигурации со скайпом и кортанами надобно пользовать другие способы раскатывания ОС на накопитель.

## Распаковка при помощи WinNTSetup3

[WinNTSetup3]

Pro:

* Разворачивание всего-всего
* Возможность установить загрузчик куда угодно (или вообще его не трогать)

Contra:

* Разбивать диск надобно вручную

Отлично запускается из WinPE. Рекомендую пользовать вместе с [BootIce].

## Создание своего образа для быстрого развёртывания
## Развёртывание из образа install.wim средствами wimlib-imagex



[NCQ]:               https://ru.wikipedia.org/wiki/NCQ
[MBR#Sector_layout]: https://en.wikipedia.org/wiki/Master_boot_record#Sector_layout

[Acronis]:           http://acronis.com/personal/disk-manager
[BootIce]: http://bbs.wuyou.net/forum.php?mod=viewthread&amp;tid=57675
[gptgen]: http://sf.net/p/gptgen	"gptgen -w \\.\\physicaldrive0"
[MiniTool]:          http://minitool.com
[Paragon]:           http://paragon.ru/home/hdm-professional
[Rufus]:             https://rufus.akeo.ie/?locale=ru_RU

[GAG]: http://sf.net/p/gag
[GRUB2WIN]: http://sf.net/p/grub2win
[grub4dos]:          http://grub4dos.chenall.net
[PLoP]: https://plop.at
[Smart BootManager]: http://sf.net/p/btmgr
[XorBoot]: http://bbs.wuyou.net/forum.php?mod=viewthread&amp;tid=157812

[Visual BCD Editor]: https://boyans.net

[efi drivers]: http://efi.akeo.ie
[EasyUEFI]: http://easyuefi.com/index-us.html
[UEFI Shell]: https://github.com/tianocore/edk2/tree/master/ShellBinPkg/UefiShell

[AHCI RAID enable]: ./files/AHCI-RAID-enable.reg
[AHCI MS solution]:  https://support.microsoft.com/en-us/kb/922976

[WinNTSetup3]:       http://msfn.org/board/topic/149612-winntsetup

[guid pic]: ./images/GUID.png
[standard pic 1]: ./images/0.1.png
[standard pic 2]: ./images/0.2.png
[standard pic 3]: ./images/0.3.png
[standard pic 4]: ./images/0.4.png