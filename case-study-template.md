# Case-study оптимизации

## Актуальная проблема
В нашем проекте возникла серьёзная проблема.

Необходимо было обработать файл с данными, чуть больше ста мегабайт.

У нас уже была программа на `ruby`, которая умела делать нужную обработку.

Она успешно работала на файлах размером пару мегабайт, но для большого файла она работала слишком долго, и не было понятно, закончит ли она вообще работу за какое-то разумное время.

Я решил исправить эту проблему, оптимизировав эту программу.

## Формирование метрики
Для того, чтобы понимать, дают ли мои изменения положительный эффект на быстродействие программы я придумал использовать такую метрику: 
изначально дождаться выполнения скрипта на больших данных не удалось, поэтому непонятно каким должен быть бюджет в плане сложности.
для начала сделаю так, чтобы данные были обработаны в течение, хотя бы, минуты.

## Гарантия корректности работы оптимизированной программы
Программа поставлялась с тестом. Выполнение этого теста в фидбек-лупе позволяет не допустить изменения логики программы при оптимизации.

## Feedback-Loop
Для того, чтобы иметь возможность быстро проверять гипотезы я выстроил эффективный `feedback-loop`, который позволил мне получать обратную связь по эффективности сделанных изменений за *время, которое у вас получилось*

Вот как я построил `feedback_loop`: сначала выделить тест в отдельный файл. запустить профилировщики, посмотреть на самые тяжеловесные вычисления,
попробовать их оптимизировать, запустить тест, проверить метрики.

## Вникаем в детали системы, чтобы найти главные точки роста
Для того, чтобы найти "точки роста" для оптимизации я воспользовался профилировщиками. rbspy установить не получилось,
какие то проблемы с зависимостями glibc на mint 20.3, stackprof выдал пустой результат,
а ruby-prof оказался полезен. особо callgrind

Вот какие проблемы удалось найти и решить

### Ваша находка №1
- Больше всего времени(21%) занимает метод collect_stats_from_users, в частности Date.parse все тормозит.
- Посмотрел входные данные - увидел, что формат даты всегда одинаковый. значит вместо Date.parse можно использовать парсер, которому не нужно угадывать формат.
  Заменил Date.parse(d) на Date.new(*d.split('-'))
- парсинг даты теперь не занимает времени совсем, и переместился с 1 места на 6, но чуть чуть прибавилось времени у split
- вроде норм, перейдем к другим проблемам

### Ваша находка №2
- теперь больше всего времени занимают IO операции, read(16%) и write(14%)
- для вывода данных выбран формат json. можно попробовать сделать это с помощью библиотеки oj. 
  для записи данных использовал Oj.to_file('result.json', report, mode: :compat)
  для чтения попробую использовать readlines
- значительных изменений нет
- IO::read осталось то же, IO::write даже увеличилось. мне кажется это нельзя изменить, поэтому нужно выбрать другую точку роста

### Ваша находка №3
- collect_stats_from_users вызывается 7 раз и занимает 33% времени. с помощью kcachegrind увидел, что в нем много раз вызывается map
- кажется, что метод много раз вызывается на одних и тех же данных. это похоже на n+1. порефакторим.
- доля array#each уменьшилась с 13% до 6, 
- незначительно уменьшил долю each в общем выполнении

### Ваша находка №4
- что если не нужно ждать окончания readlines, после которого выполнять обработку данных, а сразу выполнять обработку?
  получается читаем строку, сразу же ее обрабатываем - не нужно будет позже подгружать пользователя и все сессии.
  когда блок сессий закончится, эти же данные можно записать в файл, но это выглядит сложно, и проблема ведь не в памяти?
  можно начать с оптимизации подсчета статистик. например убрать .all? , использовать set для проверки уникальности браузеров
- почему-то во flat.txt не указывается IO::write. время выполнения уменьшилось на 300мс. но тк из 500 прошлых 24% занимал IO::write, а это около 100,
  то можно считать, что улучшилось на 200. set реально работает

### Ваша находка №5
  - помимо метода readlines оставшееся время съедает некий Array.each... их в общем то в коде много, надо подумать как уменьшить проходы по спискам
  - each это линейная операция, значит нужно взять что-то лучше. бинпоиск не поможет тк сначала данные нужно отсортировать, у нас нет на это времени.
    можно взять хеш, у него время доступа O(1). осталось придумать только в каких местах можно его применить.
    на первый взгляд можно убрать отдельный цикл sessions.each для подсчета браузеров. сделать это внутри readlines
  - доля времени write выросла, each уменьшилась. большой файл все еще не читается

### Ваша находка №6
  - в цикле users.each происходит еще один цикл users.select. это я глазом заметил и тут я начинаю подозревать, что ruby_prof мне выдает просто рандомные числа,
    потому что за последние несколько раз метрики то пропадали, но давали странные значения. и у всех абсолютное время 0
  - формирование сессий можно вынести так же в цикл чтения. заменил список sessions на хеш. тогда нам вообще не нужен класс user и список users_objects
  - как изменилась метрика? а я не понял, потому что она не изменилась, но теперь большой файл возможно обработать. надо мне пошаманить с профилировщиками
    единственное полезное на что мне указал ruby_prof это Date.parse, а то что Array.each занимает много времени мне ни о чем не говорит

## Результаты
В результате проделанной оптимизации наконец удалось обработать файл с данными.
Удалось улучшить метрику системы с того, что большой файл обработать не получалось, до того, что получилось ¯\_(ツ)_/¯

*Какими ещё результами можете поделиться*

## Защита от регрессии производительности
Для защиты от потери достигнутого прогресса при дальнейших изменениях программы добавил тест, проверяющий выполняется ли программа менее чем за минуту.
сначала думал что норм тест, а потом подождал - не норм. представил проект, в котором все тесты такие. надо подумать как уменьшить объем данных
чисто для теста. но это попозже, сейчас я устал
