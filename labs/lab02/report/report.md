---
## Front matter
title: "Лабораторная работа №2"
subtitle: "Имитационное моделирование"
author: "Серёгина Ирина Андреевна"

## Generic otions
lang: ru-RU
toc-title: "Содержание"

## Bibliography
bibliography: bib/cite.bib
csl: pandoc/csl/gost-r-7-0-5-2008-numeric.csl

## Pdf output format
toc: true # Table of contents
toc-depth: 2
lof: true # List of figures
lot: true # List of tables
fontsize: 12pt
linestretch: 1.5
papersize: a4
documentclass: scrreprt
## I18n polyglossia
polyglossia-lang:
  name: russian
  options:
	- spelling=modern
	- babelshorthands=true
polyglossia-otherlangs:
  name: english
## I18n babel
babel-lang: russian
babel-otherlangs: english
## Fonts
mainfont: IBM Plex Serif
romanfont: IBM Plex Serif
sansfont: IBM Plex Sans
monofont: IBM Plex Mono
mathfont: STIX Two Math
mainfontoptions: Ligatures=Common,Ligatures=TeX,Scale=0.94
romanfontoptions: Ligatures=Common,Ligatures=TeX,Scale=0.94
sansfontoptions: Ligatures=Common,Ligatures=TeX,Scale=MatchLowercase,Scale=0.94
monofontoptions: Scale=MatchLowercase,Scale=0.94,FakeStretch=0.9
mathfontoptions:
## Biblatex
biblatex: true
biblio-style: "gost-numeric"
biblatexoptions:
  - parentracker=true
  - backend=biber
  - hyperref=auto
  - language=auto
  - autolang=other*
  - citestyle=gost-numeric
## Pandoc-crossref LaTeX customization
figureTitle: "Рис."
tableTitle: "Таблица"
listingTitle: "Листинг"
lofTitle: "Список иллюстраций"
lotTitle: "Список таблиц"
lolTitle: "Листинги"
## Misc options
indent: true
header-includes:
  - \usepackage{indentfirst}
  - \usepackage{float} # keep figures where there are in the text
  - \floatplacement{figure}{H} # keep figures where there are in the text
---

# Цель работы

Ознакомление с протоколом TCP и алгоритмом управления очередью RED, приобретение практических навыков использования.

# Задание

1. Выполнить пример с дисциплиной RED
2. Выполнить упражнение

# Теоретическое введение

Протокол управления передачей (Transmission Control Protocol, TCP) имеет средства управления потоком и коррекции ошибок, ориентирован на установление
соединения. Объект мониторинга очереди оповещает диспетчера очереди о поступлении пакета.
Диспетчер очереди осуществляет мониторинг очереди.

# Выполнение лабораторной работы

Для начала создаю новую директорию, где буду выполнять лабораторную работу, а также файл, в котором буду прописывать сценарий (рис. [-@fig:001]).

![Создание директории](image/1.png){#fig:001 width=70%}

Требуется приписываю сценарий, реализующий модель согласно установленным требованиям, построить в Xgraph график изменения TCP-окна, график изменения длины очереди
и средней длины очереди.

```
# создание объекта Simulator
set ns [new Simulator]

# открытие на запись файла out.nam для визуализатора nam
set nf [open out.nam w]

# все результаты моделирования будут записаны в переменную nf
$ns namtrace-all $nf

# открытие на запись файла трассировки out.tr
# для регистрации всех событий
set f [open out.tr w]
# все регистрируемые события будут записаны в переменную f
$ns trace-all $f

# Процедура finish:
proc finish {} {
  global tchan_
  # подключение кода AWK:
  set awkCode {
    {
      if ($1 == "Q" && NF>2) {
        print $2, $3 >> "temp.q";
        set end $2
      }
      else if ($1 == "a" && NF>2)
        print $2, $3 >> "temp.a";
    }
  }
  set f [open temp.queue w]
  puts $f "TitleText: red"
  puts $f "Device: Postscript"
  if { [info exists tchan_] } {
    close $tchan_ 
  }
  exec rm -f temp.q temp.a
  exec touch temp.a temp.q
  exec awk $awkCode all.q
  puts $f \"queue
  exec cat temp.q >@ $f
  puts $f \n\"ave_queue
  exec cat temp.a >@ $f
  close $f
  # Запуск xgraph с графиками окна TCP и очереди:
  exec xgraph -bb -tk -x time -t "TCPRenoCWND" WindowVsTimeReno &
  exec xgraph -bb -tk -x time -y queue temp.queue &
  exit 0
}

# Формирование файла с данными о размере окна TCP:
proc plotWindow {tcpSource file} {
  global ns
  set time 0.01
  set now [$ns now]
  set cwnd [$tcpSource set cwnd_]
  puts $file "$now $cwnd"
  $ns at [expr $now+$time] "plotWindow $tcpSource $file"
}

# Узлы сети:
set N 5
for {set i 1} {$i < $N} {incr i} {
  set node_(s$i) [$ns node]
}
set node_(r1) [$ns node]
set node_(r2) [$ns node]
# Соединения:
$ns duplex-link $node_(s1) $node_(r1) 10Mb 2ms DropTail
$ns duplex-link $node_(s2) $node_(r1) 10Mb 3ms DropTail
$ns duplex-link $node_(r1) $node_(r2) 1.5Mb 20ms RED
$ns queue-limit $node_(r1) $node_(r2) 25
$ns queue-limit $node_(r2) $node_(r1) 25
$ns duplex-link $node_(s3) $node_(r2) 10Mb 4ms DropTail
$ns duplex-link $node_(s4) $node_(r2) 10Mb 5ms DropTail

# Агенты и приложения:
set tcp1 [$ns create-connection TCP/Reno $node_(s1) TCPSink $node_(s3) 0]
$tcp1 set window_ 15
set tcp2 [$ns create-connection TCP/Reno $node_(s2) TCPSink $node_(s3) 1]
$tcp2 set window_ 15
set ftp1 [$tcp1 attach-source FTP]
set ftp2 [$tcp2 attach-source FTP]

# Мониторинг размера окна TCP:
set windowVsTime [open WindowVsTimeReno w]
set qmon [$ns monitor-queue $node_(r1) $node_(r2) [open qm.out w] 0.1];
[$ns link $node_(r1) $node_(r2)] queue-sample-timeout;
# Мониторинг очереди:
set redq [[$ns link $node_(r1) $node_(r2)] queue]
set tchan_ [open all.q w]
$redq trace curq_
$redq trace ave_
$redq attach $tchan_


# Добавление at-событий:
$ns at 0.0 "$ftp1 start"
$ns at 1.1 "plotWindow $tcp1 $windowVsTime"
$ns at 3.0 "$ftp2 start"
$ns at 10 "finish"

# запуск модели
$ns run
```

Я запустила код и получила график изменения TCP окна (рис. [-@fig:002]), график изменения длины очереди со средним показателем очереди (рис. [-@fig:003]).

![График изменения TCP окна](image/2.png){#fig:002 width=70%}

![График динамики длины очереди](image/3.png){#fig:003 width=70%}

Как видно на графике, средняя длины очереди колеблется в промежутке между 2 и 4, ее максимальная длина примерно равна 13,5.

При выполнении упражнения требовалось сменить тип Reno на Newreno (рис. [-@fig:007]).

![Вношу изменения в код](image/7.png){#fig:007 width=70%}

Получаю график изменения TCP окна (рис. [-@fig:015]) и изменения длины очереди (рис. [-@fig:008]).

![График изменения TCP окна для Newreno](image/15.png){#fig:015 width=70%}

![График динамики длины очереди для Reno](image/8.png){#fig:008 width=70%}

Графики сильно схожи с теми, что я получила раньше, окно увеличивается вплоть до момента потери пакетов.

Теперь поменяю Newreno на Vegas (рис. [-@fig:009]).

![Вношу изменения в код](image/9.png){#fig:009 width=70%}

Получаю график изменения TCP окна (рис. [-@fig:016]) и изменения длины очереди (рис. [-@fig:010]).

![График изменения TCP окна для Vegas](image/16.png){#fig:016 width=70%}

![График динамики длины очереди для Vegas](image/10.png){#fig:010 width=70%}

Здесь можно заметить, что средняя длина очереди находится все ближе к значению 3, все еще не выходя за границы 2 и 4, максимальная длина очереди все та же. 
На графике размера окна видно, что его максимальный размер меньше, чем у предыдущих, 20, а не 34, то есть Vegas обнаруживает перегрузку до потери пакета и уменьшает размер окна.

Теперь поработаю с изменением отображения окон. Внесу изменения при отображении окон с графиками (изменю цвет фона,
цвет траекторий, подписи к осям, подпись траектории в легенде).

Меняю цвет графиков (рис. [-@fig:011]).

![Изменение цвета графиков](image/11.png){#fig:011 width=70%}

Меняю легенды (рис. [-@fig:012]).

![Изменение легенд](image/12.png){#fig:012 width=70%}

Затем меняю цвет фона и название оси очереди (рис. [-@fig:013]).

![Изменение фона и названия осей](image/13.png){#fig:013 width=70%}

Так выглядит график, в который были внесены изменения (рис. [-@fig:014]).

![Измененный график](image/14.png){#fig:014 width=70%}


# Выводы

Я ознакомилась с протоколом TCP и алгоритмом управления очередью RED, приобрела практические навыки использования.


