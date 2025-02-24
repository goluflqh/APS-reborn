# Лабораторная работа 13 "Высокоуровневое программирование"

## Цель

В соответствии с индивидуальным заданием, написать программу на языке программирования высокого уровня C, скомпилировать в машинные коды и запустить на ранее разработанном процессоре RISC-V.

## Ход работы

1. Изучить теорию:
   1. [Соглашение о вызовах](#соглашение-о-вызовах)
   2. [Скрипт для компоновки](#скрипт-для-компоновки-linker_scriptld)
   3. [Файл первичных команд](#файл-первичных-команд-при-загрузке-startups)
2. [Подготовить набор инструментов для кросс-компиляции](#практика)
3. Изучить порядок компиляции и команды, её осуществляющую:
   1. [Компиляция объектных файлов](#компиляция-объектных-файлов)
   2. [Компоновка объектных файлов в исполняемый](#компоновка-объектных-файлов-в-исполняемый)
   3. [Экспорт секций для инициализации памяти](#экспорт-секций-для-инициализации-памяти)
   4. [Дизассемблирование](#дизассемблирование)
4. [Написать и скомпилировать собственную программу](#задание).
5. Проверить исполнение программы вашим процессором в ПЛИС.

## Теория

В рамках данной лабораторной работы вы напишите полноценную программу, которая будет запущена на вашем процессоре. В процессе компиляции, вам потребуются файлы [linker_script.ld](linker_script.ld) и [startup.S](startup.S), лежащие в этой папке.

> — Но зачем мне эти файлы? Мы ведь уже делали задания по программированию на предыдущих лабораторных работах и нам не были нужны никакие дополнительные файлы.

Дело в том, что ранее вы писали небольшие программки на ассемблере. Однако, язык ассемблера архитектуры RISC-V, так же как и любой другой RISC архитектуры, недружелюбен к программисту, поскольку изначально создавался с прицелом на то, что будут созданы компиляторы и программы будут писаться на более удобных для человека языках высокого уровня. Ранее вы писали простенькие программы, которые можно было реализовать на ассемблере, теперь же вам будет предложено написать полноценную программу на языке Си.

> —  Но разве в процессе компиляции исходного кода на языке Си мы не получаем программу, написанную на языке ассемблера? Получится ведь тот же код, что мы могли написать и сами.

Штука в том, что ассемблерный код который писали ранее вы отличается от ассемблерного кода, генерируемого компилятором. Код, написанный вами обладал, скажем так... более тонким микро-контролем хода программы. Когда вы писали программу, вы знали какой у вас размер памяти, где в памяти расположены инструкции, а где данные (ну, при написании программ вы почти не пользовались памятью данных, а когда пользовались — просто лупили по случайным адресам и все получалось). Вы пользовались всеми регистрами регистрового файла по своему усмотрению, без ограничений. Однако, представьте на секунду, что вы пишете проект на ассемблере вместе с коллегой: вы пишите одни функции, а он другие. Как в таком случае вы будете пользоваться регистрами регистрового файла? Поделите его напополам и будете пользоваться каждый своей половиной? Но что будет, если к проекту присоединится еще один коллега — придется делить регистровый файл уже на три части? Так от него уже ничего не останется. Для разрешения таких ситуаций было разработано [соглашение о вызовах](#соглашение-о-вызовах) (calling convention).

Таким образом, генерируя ассемблерный код, компилятор не может так же, как это делали вы, использовать все ресурсы без каких-либо ограничений — он должен следовать ограничениям, накладываемым на него соглашением о вызовах, а так же ограничениям, связанным с тем, что он ничего не знает о памяти устройства, в котором будет исполняться программа — а потому он не может работать с памятью абы как. Работая с памятью, компилятор следует некоторым правилам, благодаря которым после компиляции компоновщик сможет собрать программу под ваше устройство с помощью специального скрипта.

### Соглашение о вызовах

Соглашение о вызовах [устанавливает](https://github.com/riscv-non-isa/riscv-elf-psabi-doc/releases/download/v1.0/riscv-abi.pdf) порядок вызова функций: где размещаются аргументы при вызове функций, где находятся указатель на стек и адрес возврата и т.п.

Кроме того, соглашение делит регистры регистрового файла на две группы: оберегаемые и необерегаемые регистры.

При работе с оберегаемыми регистрами, функция должна гарантировать, что перед возвратом в этих регистрах останется тоже самое значение, что было при вызове функции. То есть, если функция собирается записать что-то в оберегаемый регистр, она должна сохранить перед этим его значение на стек, а затем, перед возвратом, вернуть это значение со стека обратно в этот же регистр.

Простая аналогия — в маленькой квартире двое делят один рабочий стол по времени. Каждый использует стол по полной, но после себя он должен оставить половину стола соседа (оберегаемые регистры) в том же виде, в котором ее получил, а со своей (необерегаемые регистры) делает что хочет. Кстати, вещи соседа, чтоб не потерять, убирают на стопку рядом (в основную память).

С необерегаемыми регистрами функция может работать как ей угодно — не существует никаких гарантий, которые вызванная функция должна исполнить. При этом, если функция вызывает другую функцию, она точно так же не получает никаких гарантий, что вызванная функция оставит значения необерегаемых регистров без изменений, поэтому если там хранятся значения, которые потребуются по окончанию выполнения вызываемой функции, эти значения необходимо сохранить на стек.

В таблице ниже приведено разделение регистров на оберегаемые (в правом столбце записано `Callee`, т.е. за их сохранение отвечает вызванная функция) и необерегаемые (`Caller` — за сохранение отвечает вызывающая функция). Кроме того, есть три регистра, для которых правый столбец не имеет значения: нулевой регистр (поскольку его невозможно изменить) и указатели на поток и глобальную область памяти. По соглашению о вызовах, эти регистры нельзя использовать для вычислений функций, они изменяются только по заранее оговоренным ситуациям.

В столбце `ABI name` записывается синоним имени регистра, связанный с его функциональным назначением (см. описание регистра). Часто ассемблеры одинаково воспринимают обе формы написания имени регистров.

|Register|ABI Name|           Description           | Saver |
|--------|--------|---------------------------------|-------|
|   x0   |  zero  |Hard-wired zero                  |   —   |
|   x1   |  ra    |Return address                   |Caller |
|   x2   |  sp    |Stack pointer                    |Callee |
|   x3   |  gp    |Global pointer                   |   —   |
|   x4   |  tp    |Thread pointer                   |   —   |
|   x5   |  t0    |Temporary/alternate link register|Caller |
|  x6–7  |  t1–2  |Temporaries                      |Caller |
|   x8   |  s0/fp |Saved register/frame pointer     |Callee |
|   x9   |  s1    |Saved register                   |Callee |
| x10–11 |  a0–1  |Function arguments/return values |Caller |
| x12–17 |  a2–7  |Function arguments               |Caller |
| x18–27 |  s2–11 |Saved registers                  |Callee |
| x28–31 |  t3–6  |Temporaries                      |Caller |

_Таблица 1. Ассемблерные мнемоники для целочисленных регистров RISC-V и их назначение в соглашении о вызовах._

Не смотря на то, что указатель на стек помечен как Callee-saved регистр, это не означает, вызываемая функция может записать в него что заблагорассудится, предварительно сохранив его значение на стек. Ведь как вы вернете значение указателя на стек со стека, если в регистре указателя на стек лежит что-то не то?

Запись `Callee` означает, что к моменту возврата из вызываемой функции, значение Callee-saved регистров должно быть ровно таким же, каким было в момент вызова функций. Для s0-s11 регистров это осуществляется путем сохранения их значений на стек. При этом, перед каждым сохранением на стек, изменяется значение указателя на стек таким образом, чтобы он указывал на сохраняемое значение (обычно он декрементируется). Затем, перед возвратом из функций все сохраненные на стек значения восстанавливаются, попутно изменяя значение указателя на стек противоположным образом (инкрементируют его). Таким образом, не смотря на то, что значение указателя на стек менялось в процессе работы вызываемой функции, к моменту выхода из нее, его значение в итоге останется тем же.

### Скрипт для компоновки (linker_script.ld)

Скрипт для компоновки описывает то, как в вашей памяти будут храниться данные. Вы уже могли слышать о том, что исполняемый файл содержит секции `.text` и `.data` — инструкций и данных соответственно. Компоновщик (**linker**) ничего не знает о том, какая у вас структура памяти: принстонская у вас архитектура или гарвардская, по каким адресам у вас должны храниться инструкции, а по каким данные, какой в памяти используется порядок следования байт (**endianess**). У вас может быть несколько типов памятей под особые секции — и обо всем этом компоновщику можно сообщить в скрипте для компоновки.

В самом простом виде скрипт компоновки состоит из одного раздела: раздела секций, в котором вы и описываете какие части программы куда и в каком порядке необходимо разместить.

Для удобства этого описания существует вспомогательная переменная: счетчик адресов. В начале скрипта этот счетчик равен нулю. Размещая очередную секцию, этот счетчик увеличивается на размер этой секции. Допустим, у нас есть два файла `fourier.o` и `main.o`, в каждом из которых есть секции `.text` и `.data`. Мы хотим разместить их в памяти следующим образом: сперва разместить секции `.text` обоих файлов, а затем секции `.data`.

В итоге начиная с нулевого адреса будет размещена секция `.text` файла `fourier.o`. Она будет размещена именно там, поскольку счетчик адресов в начале скрипта равен нулю, а очередная секция размещается по адресу, куда указывает счетчик адресов. После этого, счетчик адресов будет увеличен на размер этой секции и секция `.text` файла `main.o` будет размещена сразу же за секцией `.text` файла `fourier.o`. После этого счетчик адресов будет увеличен на размер этой секции. То же самое произойдет и при размещении оставшихся секций.

Кроме того, вы в любой момент можете изменить значение счетчика адресов. Например, у вас две раздельные памяти: память инструкций объемом 512 байт и память данных объемом 1024 байта. Эти памяти находятся в одном адресном пространстве. Диапазон адресов памяти инструкций: `[0:511]`, диапазон памяти данных: `[512:1535]`. При этом общий объем секций `.text` составляет 416 байт. В этом случае, вы можете сперва разместить секции `.text` так же, как было описано в предыдущем примере, а затем, выставив значение на счетчике адресов равное `512`, описываете размещение секций данных. Тогда, между секциями будет появится разрыв в 96 байт. А данные окажутся в диапазоне адресов, выделенном для памяти данных.

Помимо прочего, в скрипте компоновщика необходимо прописать где будет находиться стек, и какое будет значение у указателя на глобальную область памяти.

Все это с подробными комментариями описано в файле `linker_script.ld`.

```ld
OUTPUT_FORMAT("elf32-littleriscv")      /* Указываем порядок следования байт */

ENTRY(_start)                           /* мы сообщаем компоновщику, что первая
                                           исполняемая процессором инструкция
                                           находится у метки "start"
                                        */

/*
  Объявляем вспомогательные глобальные переменные
*/
_text_size        = 0x4000;              /* Размер памяти инстр.: 16KiB */
_data_base_addr   = 0x8000;              /* Стартовый адрес секции данных */
_data_size        = 0x4000;              /* Размер памяти данных: 16KiB */

_data_end = _data_base_addr + _data_size;

_trap_stack_size  = 2560;                /* Размер стека обработчика перехватов.
                                           Данный размер позволяет выполнить
                                           до 32 вложенных вызовов при обработке
                                           перехватов.
                                         */

_stack_size       = 1280;                /* Размер программного стека.
                                           Данный размер позволяет выполнить
                                           до 16 вложенных вызовов.
                                         */
SECTIONS
{
  PROVIDE( _start = 0x00000000 );       /* Позиция start в памяти
  /*
    В скриптах компоновщика есть внутренняя переменная, записываемая как '.'
    Эта переменная называется "счетчиком адресов". Она хранит текущий адрес в
    памяти.
    В начале файла она инициализируется нулем. Добавляя новые секции, эта
    переменная будет увеличиваться на размер каждой новой секции.
    Если при размещении секций не указывается никакой адрес, они будут размещены
    по текущему значению счетчика адресов.
    Этой переменной можно присваивать значения, после этого, она будет
    увеличиваться с этого значения.
    Подробнее:
      https://home.cs.colorado.edu/~main/cs1300/doc/gnu/ld_3.html#IDX338
  */

  /*
    Следующая команда сообщает, что начиная с адреса, которому в данных момент
    равен счетчик адресов (в данный момент, начиная с нуля) будет находиться
    секция .text итогового файла, которая состоит из секций .boot, а также всех
    секций, начинающихся на .text во всех переданных компоновщику двоичных
    файлах.
  */
  .text : {*(.boot) *(.text*)}


  /*
    Поскольку мы не знаем суммарного размера получившейся секции, мы проверяем
    что не вышли за границы памяти инструкций и переносим счетчик адресов за пределы. Памяти инструкций в область памяти данных
  */
  ASSERT(. < _text_size, ".text section exceeds instruction memory size")
  . = _data_base_addr;

  /*
    Следующая команда сообщает, что начиная с адреса, которому в данных момент
    равен счетчик адресов (_data_base_addr) будет находиться секция .data итогового файла, которая состоит из секций всех секций, начинающихся
    на .data во всех переданных компоновщику двоичных файлах.
  */
  .data : {*(.data*)}

  /*
    Общепринято присваивать GP значение равное началу секции данных, смещенное
    на 2048 байт вперед.
    Благодаря относительной адресации со смещением в 12 бит, можно адресоваться
    на начало секции данных, а так же по всему адресному пространству вплоть до
    4096 байт от начала секции данных, что сокращает объем требуемых для
    адресации инструкций (практически не используются операции LUI, поскольку GP
    уже хранит базовый адрес и нужно только смещение).
    Подробнее:
      https://groups.google.com/a/groups.riscv.org/g/sw-dev/c/60IdaZj27dY/m/s1eJMlrUAQAJ
  */
  _gbl_ptr = _data_base_addr + 0x800;


  /*
    Поскольку мы не знаем суммарный размер всех используемых секций данных,
    перед размещением других секций, необходимо выравнять счетчик адресов по 4х-байтной границе.
  */
  . = ALIGN(4);


  /*
    BSS (block started by symbol, неофициально его расшифровывают как
    better save space) — это сегмент, в котором размещаются неинициализированные
    статические переменные. В стандарте Си сказано, что такие переменные
    инициализируются нулем (или NULL для указателей). Когда вы создаете
    статический массив — он должен быть размещен в исполняемом файле.
    Без bss-секции, этот массив должен был бы занимать такой же объем
    исполняемого файла, какого объема он сам. Массив на 1000 байт занял бы
    1000 байт в секции .data.
    Благодаря секции bss, начальные значения массива не задаются, вместо этого
    здесь только записываются названия переменных и их адреса.
    Однако на этапе загрузки исполняемого файла теперь необходимо принудительно
    занулить участок памяти, занимаемый bss-секцией, поскольку статические
    переменные должны быть проинициализированы нулем.
    Таким образом, bss-секция значительным образом сокращает объем исполняемого
    файла (в случае использования неинициализированных статических массивов)
    ценой увеличения времени загрузки этого файла.
    Для того, чтобы занулить bss-секцию, в скрипте заводятся две переменные,
    указывающие на начало и конец bss-секции посредством счетчика адресов.
    Подробнее:
      https://en.wikipedia.org/wiki/.bss
  */
  _bss_start = .;
  .bss : {*(.bss*)}
  _bss_end = .;


  /*=================================
      Секция аллоцированных данных завершена, остаток свободной памяти отводится
      под программный стек, стек прерываний и (возможно) кучу. В соглашении о
      вызовах архитектуры RISC-V сказано, что стек растет снизу вверх, поэтому
      наша цель разместить его в самых последних адресах памяти.
      Поскольку стеков у нас два, в самом низу мы разместим стек прерываний, а
      над ним программный стек. При этом надо обеспечить защиту программного
      стека от наложения на него стека прерываний.
      Однако перед этим, мы должны убедиться, что под программный стек останется
      хотя бы 1280 байт (ничем не обоснованное число, взятое с потолка).
      Такое значение обеспечивает до 16 вложенных вызовов (если сохранять только необерегаемые регистры).
    =================================
  */

  /* Мы хотим гарантировать, что под стек останется как минимум 1280 байт */
  ASSERT(. < (_data_end - _trap_stack_size - _stack_size),
            "Program size is too big")

  /*  Перемещаем счетчик адресов над стеком прерываний (чтобы после мы могли
      использовать его в вызове ALIGN) */
  . = _data_end - _trap_stack_size;

  /*
      Размещаем указатель программного стека так близко к границе стека
      прерываний, насколько можно с учетом требования о выравнивании адреса
      стека до 16 байт.
      Подробнее:
        https://riscv.org/wp-content/uploads/2015/01/riscv-calling.pdf
  */
  _stack_ptr = ALIGN(16) <= _data_end - _trap_stack_size?
                ALIGN(16) : ALIGN(16) - 16;
  ASSERT(_stack_ptr <= _data_end - _trap_stack_size, "SP exceed memory size")

  /*  Перемещаем счетчик адресов в конец памяти (чтобы после мы могли
      использовать его в вызове ALIGN) */
  . = _data_end;

  /*
      Обычно память имеет размер, кратный 16, но на случай, если это не так, мы
      делаем проверку, после которой мы либо остаемся в самом конце памяти (если
      конец кратен 16), либо поднимаемся на 16 байт вверх от края памяти,
      округленного до 16 в сторону большего значения
  */
  _trap_stack_ptr = ALIGN(16) <= _data_end ? ALIGN(16) : ALIGN(16) - 16;
  ASSERT(_trap_stack_ptr <= _data_end, "ISP exceed memory size")
}
```

_Листинг 1. Пример скрипта компоновщика с комментариями._

### Файл первичных команд при загрузке (startup.S)

В стартап-файле хранятся инструкции, которые обязательно необходимо выполнить перед началом исполнения любой программы. Это инициализация регистров указателей на стек и глобальную область данных, регистра, хранящего адрес вектора прерываний и т.п.

По завершению инициализации, стартап-файл выполняет процедуру передаче управления точке входа в запускаемую программу.

```asm
  .section    .boot

 .global _start
_start:
  la    gp, _gbl_ptr     # Инициализация глобального указателя
  la    sp, _stack_ptr   # Инициализация указателя на стек

# Инициализация (зануление) сегмента bss
  la    t0, _bss_start
  la    t1, _bss_end
_bss_init_loop:
  beq   t0, t1, _irq_config
  sw    zero, 0(t0)
  addi  t0, t0, 4
  j     _bss_init_loop

# Настройка вектора (mtvec) и маски (mie) прерываний, а также указателя на стек
# прерываний (mscratch).
_irq_config:
  la    t0, _int_handler
  li    t1, -1 # -1 (все биты равны 1) означает, что разрешены все прерывания
  la    t2, _trap_stack_ptr
  csrw  mtvec, t0
  csrw  mie, t1
  csrw  mscratch, t2

# Вызов функции main
_main_call:
  li    a0, 0 # Передача аргументов argc и argv в main. Формально, argc должен
  li    a1, 0 # быть больше нуля, а argv должен указывать на массив строк,
              # нулевой элемент которого является именем исполняемого файла,
              # Но для простоты реализации оба аргумента всего лишь обнулены.
              # Это сделано для детерминированного поведения программы в случае,
              # если будет пытаться использовать эти аргументы.
  call  main
# Зацикливание после выхода из функции main
_endless_loop:
  j     _endless_loop

# Низкоуровневый обработчик прерывания отвечает за:
#   * Сохранение и восстановление контекста;
#   * Вызов высокоуровневого обработчика с передачей id источника прерывания в
#     качестве аргумента.
# В основе кода лежит обработчик из репозитория urv-core:
# https://github.com/twlostow/urv-core/blob/master/sw/common/irq.S
# Из реализации убраны сохранения нереализованных CS-регистров. Кроме того,
# судя по документу приведенному ниже, обычное ABI подразумевает такое же
# сохранение контекста, что и при программном вызове (EABI подразумевает еще
# меньшее сохранение контекста), поэтому нет нужды сохранять весь регистровый
# файл.
# Документ:
#  https://github.com/riscv-non-isa/riscv-eabi-spec/blob/master/EABI.adoc
_int_handler:
  # Данная операция меняет местами регистры sp и mscratch.
  # В итоге указатель на стек прерываний оказывается в регистре sp, а вершина
  # программного стека оказывается в регистре mscratch.
  csrrw sp,mscratch,sp

  # Далее мы поднимаемся по стеку прерываний и сохраняем все регистры.
  addi  sp,sp,-80  # Указатель на стек должен быть выровнен до 16 байт, поэтому
                    # поднимаемся вверх не на 76, а на 80.
  sw    ra,4(sp)
  # Мы хотим убедиться, что очередное прерывание не наложит стек прерываний на
  # программный стек, поэтому записываем в освободившийся регистр низ
  # программного стека, и проверяем что приподнятый указатель на верхушку
  # стека прерываний не залез в программный стек.
  # В случае, если это произошло (произошло переполнение стека прерываний),
  # мы хотим остановить работу процессора, чтобы не потерять данные, которые
  # могут помочь нам в отладке этой ситуации.
  la    ra, _stack_ptr
  blt   sp, ra, _endless_loop

  sw    t0,12(sp) # Мы перепрыгнули через смещение 8, поскольку там должен
                  # лежать регистр sp, который ранее сохранили в mscratch.
                  # Мы запишем его на стек чуть позже.
  sw    t1,16(sp)
  sw    t2,20(sp)
  sw    a0,24(sp)
  sw    a1,28(sp)
  sw    a2,32(sp)
  sw    a3,36(sp)
  sw    a4,40(sp)
  sw    a5,44(sp)
  sw    a6,48(sp)
  sw    a7,52(sp)
  sw    t3,56(sp)
  sw    t4,60(sp)
  sw    t5,64(sp)
  sw    t6,68(sp)

  # Кроме того, мы сохраняем состояние регистров прерываний на случай, если
  # произойдет еще одно прерывание.
  csrr  t0,mscratch
  csrr  t1,mepc
  csrr  a0,mcause
  sw    t0,8(sp)
  sw    t1,72(sp)
  sw    a0,76(sp)

  # Вызов высокоуровневого обработчика прерываний
  call  int_handler

  # Восстановление контекста. В первую очередь мы хотим восстановить CS-регистры,
  # на случай, если происходило вложенное прерывание. Для этого, мы должны
  # вернуть исходное значение указателя стека прерываний. Однако его нынешнее
  # значение нам еще необходимо для восстановления контекста, поэтому мы
  # сохраним его в регистр a0, и будем восстанавливаться из него.
  mv    a0,sp

  lw    t1,72(a0)
  addi  sp,sp,80
  csrw  mscratch,sp
  csrw  mepc,t1
  lw    ra,4(a0)
  lw    sp,8(a0)
  lw    t0,12(a0)
  lw    t1,16(a0)
  lw    t2,20(a0)
  lw    a1,28(a0)   # Мы пропустили a0, потому что сейчас он используется в
                    # качестве указателя на верхушку стека и не может быть
                    # восстановлен.
  lw    a2,32(a0)
  lw    a3,36(a0)
  lw    a4,40(a0)
  lw    a5,44(a0)
  lw    a6,48(a0)
  lw    a7,52(a0)
  lw    t3,56(a0)
  lw    t4,60(a0)
  lw    t5,64(a0)
  lw    t6,68(a0)
  lw    a0,40(a0)

  # Выход из обработчика прерывания
  mret
```

_Листинг 2. Пример содержимого файла первичных команд с поясняющими комментариями._

## Практика

Для того, чтобы запустить моделирование исполнения программы на вашем процессоре, сперва эту программу необходимо скомпилировать и преобразовать в текстовый файл, которым САПР сможет проинициализировать память процессора. Для компиляции программы, вам потребуется особый компилятор, который называется "кросскомпилятор". Он позволяет компилировать исходный код под архитектуру компьютера, отличную от компьютера, на котором ведется компиляция. В нашем случае, вы будете собирать код под архитектуру RISC-V на компьютере с архитектурой `x86_64`.

Компилятор, который подойдет для данной задачи (для запуска в операционной системе Windows) уже установлен в аудиториях. Но если что, вы можете скачать его [отсюда](https://github.com/xpack-dev-tools/riscv-none-elf-gcc-xpack/releases/download/v13.2.0-1/xpack-riscv-none-elf-gcc-13.2.0-1-win32-x64.zip).

### Компиляция объектных файлов

В первую очередь необходимо скомпилировать файлы с исходным кодом в объектные. Это можно сделать следующей командой:

```text
<исполняемый файл компилятора> -с <флаги компиляции> <входной файл с исходным кодом> -o <выходной объектный файл>
```

Вам потребуются следующие флаги компиляции:

* `-march=rv32i_zicsr` — указание разрядности и набора расширений в архитектуре, под которую идет компиляция (у нас процессор rv32i с расширением инструкциями для взаимодействия с регистрами контроля и статуса Zicsr)
* `-mabi=ilp32`  — указание двоичного интерфейса приложений. Здесь сказано, что и типы `int`, `long` и `pointer` являются 32-разрядными.

Есть очень [хорошее видео](https://youtu.be/29iNHEhHmd0?t=141), описывающее состав тулчейнов, именование исполняемых файлов компиляторов, как формируются ключи архитектуры и двоичного интерфейса приложений.

С учетом названия исполняемого файла скачанного вами компилятора (при условии, что папку из архива вы переименовали в `riscv_cc` и скопировали в корень диска `C:`, а команду запускаете из терминала `git bash`), командой для компиляции файла [`startup.S`](startup.S) может быть:

```bash
/c/riscv_cc/bin/riscv-none-elf-gcc -c -march=rv32i_zicsr -mabi=ilp32 startup.S -o startup.o
```

### Компоновка объектных файлов в исполняемый

Далее необходимо выполнить компоновку объектных файлов. Это можно выполнить командной следующего формата:

```text
<исполняемый файл компилятора> <флаги компоновки> <входные объектные файлы> -o <выходной объектный файл>
```

Исполняемый файл компилятора тот же самый, флаги компоновки будут следующие:

* `-march=rv32i_zicsr -mabi=ilp32` — те же самые флаги, что были при компиляции (нам все еще нужно указывать архитектуру, иначе компоновщик может скомпоновать объектные файлы со стандартными библиотеками от другой архитектуры)
* `-Wl,--gc-sections` — указать компоновщику удалять неиспользуемые секции (сокращает объем итогового файла)
* `-nostartfiles` — указать компоновщику не использовать стартап-файлы стандартных библиотек (сокращает объем файла и устраняет ошибки компиляции из-за конфликтов с используемым стартап-файлом).
* `-T linker_script.ld` — передать компоновщику скрипт компоновки

Пример команды компоновки:

```bash
/c/riscv_cc/bin/riscv-none-elf-gcc -march=rv32i_zicsr -mabi=ilp32 -Wl,--gc-sections -nostartfiles -T linker_script.ld startup.o main.o -o result.elf
```

### Экспорт секций для инициализации памяти

В результате компоновки вы получите исполняемый файл формата `elf` ([Executable and Linkable Format](https://ru.wikipedia.org/wiki/Executable_and_Linkable_Format)). Это двоичный файл, однако это не просто набор двоичных инструкций и данных, которые будут загружены в память процессора. Данный файл содержит заголовки и специальную информацию, которая поможет загрузчику разместить этот файл в памяти компьютера. Поскольку роль загрузчика будете выполнять вы и САПР, на котором будет вестись моделирование, эта информация вам не понадобятся, поэтому вам потребуется экспортировать из данного файла только двоичные инструкции и данные, отбросив всю остальную информацию. Полученный файл уже можно будет использовать в функции `$readmemh`.

Для экспорта используйте команду:

```bash
/c/riscv_cc/bin/riscv-none-elf-objcopy -O verilog result.elf init.mem
```

ключ `-O verilog` говорит о том, что файл надо сохранить в формате, который сможет воспринять команда `$readmemh`.

Поскольку память инструкций и данных у вас разделены, можно экспортировать отдельные секции в разные файлы:

```bash
/c/riscv_cc/bin/riscv-none-elf-objcopy -O verilog -j .text result.elf init_instr.mem
/c/riscv_cc/bin/riscv-none-elf-objcopy -O verilog -j .data -j .bss -j .sdata result.elf init_data.mem
```

Обратите внимание на содержимое полученного файла:

```text
@00000000
97 11 00 00 93 81 01 AD 13 01 00 76 93 02 00 2D
13 03 00 2D 63 88 62 00 23 A0 02 00 93 82 42 00
6F F0 5F FF 93 02 40 04 13 03 F0 FF 73 90 52 30
73 10 43 30 13 05 00 00 93 05 00 00 EF 00 C0 1F
6F 00 00 00 73 11 01 34 13 01 01 FB 23 22 11 00
...
```

Первая строка говорит о том, что память необходимо инициализировать с нулевого адреса и в данный момент нас не интересует. Важно то, что файл был экспортирован побайтово. Такой формат файла не подойдет для нашей памяти, т.к. каждая ячейка нашей памяти состоит из 4х байт.

Для того, чтобы итоговый файл подходил для памяти с 32-разрядными ячейками, команду экспорта необходимо дополнить опцией `--verilog-data-width=4`, которая указывает размер ячейки инициализируемой памяти в байтах. Файл примет следующий вид:

```text
@00000000
00001197 AD018193 76000113 2D000293
2D000313 00628863 0002A023 00428293
FF5FF06F 04400293 FFF00313 30529073
30431073 00000513 00000593 1FC000EF
0000006F 34011173 FB010113 00112223
...
```

Обратите внимание что байты не просто склеились в четверки, изменился так же и порядок следования байт. Это важно, т.к. в память данные должны лечь именно в таком (обновленном) порядке байт (см. первую строчку скрипта компоновщика). Когда-то `objcopy` содержал [баг](https://sourceware.org/bugzilla/show_bug.cgi?id=25202), из-за которого порядок следования байт не менялся. В каких-то версиях тулчейна (отличных от представленного в данной лабораторной работе) вы все еще можете столкнуться с подобным поведением.

Вернемся к первой строке: `@00000000`. Как уже говорилось, число, начинающееся с символа `@` говорит САПР, что с этого момента инициализация идет начиная с ячейки памяти, номер которой совпадает с этим числом. Когда вы будете экспортировать секции данных, первой строкой будет: `@00001000`. Так произойдет, поскольку в скрипте компоновщика сказано, что секция данных идет с `0x00004000` адреса. Это было сделано, чтобы не произошло наложения адресов памяти инструкций и памяти данных. **Чтобы система работала корректно, эту строчку необходимо удалить.**

### Дизассемблирование

В процессе отладки лабораторной работы потребуется много раз смотреть на программный счетчик и текущую инструкцию. Довольно тяжело декодировать инструкцию самостоятельно, чтобы понять что сейчас выполняется. Для облегчения задачи можно дизасемблировать скомпилированный файл. Полученный файл на языке ассемблера будет хранить адреса инструкций, а так же их двоичное и ассемблерное представление.

Пример дизасемблированного файла:

```asm
Disassembly of section .text:

00000000 <_start>:
   0: 00001197           auipc gp,0x1
   4: adc18193           addi gp,gp,-1316 # adc <_gbl_ptr>
   8: 76000113           li sp,1888
   c: 2dc00293           li t0,732
  10: 2dc00313           li t1,732

00000014 <_bss_init_loop>:
  14: 00628863           beq t0,t1,24 <_irq_config>
  18: 0002a023           sw zero,0(t0)
  1c: 00428293           addi t0,t0,4
...

00000164 <bubble_sort>:
 164: fd010113           addi sp,sp,-48
 168: 02112623           sw ra,44(sp)
 16c: 02812423           sw s0,40(sp)
 170: 03010413           addi s0,sp,48
 174: fca42e23           sw a0,-36(s0)
 178: fcb42c23           sw a1,-40(s0)
 17c: fe042623           sw zero,-20(s0)
 180: 09c0006f           j 21c <bubble_sort+0xb8>
...

00000244 <main>:
 244: ff010113           addi sp,sp,-16
 248: 00112623           sw ra,12(sp)
 24c: 00812423           sw s0,8(sp)
 250: 01010413           addi s0,sp,16
 254: 00a00593           li a1,10
 258: 2b400513           li a0,692
 25c: f09ff0ef           jal ra,164 <bubble_sort>
 260: 2b400793           li a5,692
...

Disassembly of section .data:

000002b4 <array_to_sort>:
 2b4: 00000003           lb zero,0(zero) # 0 <_start>
 2b8: 0005                 c.nop 1
 2ba: 0000                 unimp
 2bc: 0010                 0x10
 2be: 0000                 unimp
...
```

Числа в самом левом столбце, увеличивающиеся на 4 — это адреса в памяти. Отлаживая программу на временной диаграмме вы можете ориентироваться на эти числа, как на значения PC.

Следующая за адресом строка, записанная в шестнадцатеричном виде — это та инструкция (или данные), которая размещена по этому адресу. С помощью этого столбца вы можете проверить, что считанная инструкция на временной диаграмме (сигнал `instr`) корректна.

В правом столбце находится ассемблерный (человекочитаемый) аналог инструкции из предыдущего столбца. Например, инструкция `00001197` — это операция `auipc gp,0x1`, где `gp` — это синоним (ABI name) регистра `x3` (см. раздел [Соглашение о вызовах](#соглашение-о-вызовах)).

Обратите внимание на последнюю часть листинга: дизасм секции `.data`. В этой секции адреса могут увеличиваться на любое число, шестнадцатеричные данные могут быть любого размера, а на ассемблерные инструкции в правом столбце и вовсе не надо обращать внимание.

Дело в том, что дизасемблер пытается декодировать вообще все двоичные данные, которые видит: не делая различий инструкции это или нет. В итоге, если у него получается как-то декодировать байты из секции данных (которые могут быть абсолютно любыми) — он это сделает. Причем получившиеся инструкции могут быть из совершенно не поддерживаемых текущим файлом расширений: сжатыми (по два байта вместо четырех), инструкциями операций над числами с плавающей точкой, атомарными и т.п.

Это не значит, что секция данных в дизасме бесполезна — в приведенном выше листинге вы можете понять, что первыми элементами массива `array_to_sort` являются числа `3`, `5`, `10`, а так же то, по каким адресам они лежат (`0x2b4`, `0x2b8`, `0x2bc`, если непонятно почему первое число записано в одну 4-байтовую строку, а два других разделены на две двубайтовые — попробуйте перечитать предыдущий абзац). Просто разбирая дизасемблерный файл, обращайте внимание на то, какую именно секцию вы сейчас читаете.

Для того, чтобы произвести дизасемблирование, необходимо выполнить следующую команду:

```text
<исполняемый файл дизасемблера> -D (либо -d) <входной исполняемый файл> > <выходной файл на языке ассемблер>
```

Для нашего примера, командной будет

```bash
/c/riscv_cc/bin/riscv-none-elf-objdump -D result.elf > disasmed_result.S
```

Опция `-D` говорит что дизасемблировать необходимо вообще все секции. Опция `-d` говорит дизасемблировать только исполняемые секции (секции с инструкциями). Таким образом, выполнив дизасемблирование с опцией `-d` мы избавимся от проблем с непонятными инструкциями, в которые декодировались данные из секции `.data`, однако в этом случае, мы не сможем проверить адреса и значения, которые хранятся в этих секциях.

---

## Задание

Вам необходимо написать программу для вашего [индивидуального задания](../04.%20Primitive%20programmable%20device/Индивидуальное%20задание#индивидуальные-задания) к 4ой лабораторной работе на языке C или C++ (в зависимости от выбранного языка необходимо использовать соответствующий компилятор: gcc для C, g++ для C++).

При этом, вам необходимо получить входные данные от вашего устройства ввода и вывести результат на устройство вывода. Продумайте, как именно будет работать ваша программа, (бесконечно пересчитывать значения, получая новые данные от устройства ввода, или считать один раз, ожидая данные в бесконечном цикле — вариантов реализации очень много).

Доступ к регистрам контроллеров периферии осуществляется через обращение в память. В простейшем случае такой доступ осуществляется через [разыменование указателей](https://ru.wikipedia.org/wiki/Указатель_(тип_данных)#Действия_над_указателями), проинициализированных адресами регистров из [карты памяти](../12.%20Peripheral%20units#задание) 12-ой лабораторной работы.

При написании программы, помните что в C++ сильно ограничена арифметика указателей, поэтому при присваивании указателю целочисленного значения адреса, необходимо использовать оператор `reinterpret_cast`.

Для того, чтобы уменьшить ваше взаимодействие черной магией указателей, вам представлен файл [platform.h](platform.h), в котором объявлены структуры, в которых происходит отображение полей на физические адреса периферийных устройств. Вам нужно лишь проинициализировать указатель на структуру физическим адресом периферийного устройства (преобразовав перед этим целочисленное значение в тип указателя с помощью макроса `CAST`, представленного в файле `platform.h`).

Пример взаимодействия с периферийным устройством через вымышленную структуру:

```C++
#include "platform.h"

/*
  Создаем указатель на структуру SUPER_COLLIDER_HANDLE и инициализируем этот
  указатель адресом 0xFF000000, поскольку в нашей системе это периферийное
  устройство расположено под номером 255.
  При инициализации указателя, мы делали преобразование типа посредством
  макроса CAST(type, address) из заголовочного файла platform.h
*/
struct SUPER_COLLIDER_HANDLE* collider_ptr = CAST(struct SUPER_COLLIDER_HANDLE*, 0xFF000000);

int main(int argc, char** argv)
{
  while(1){                         // В бесконечном цикле
    while (!(collider_ptr->ready)); // Постоянно опрашиваем регистр ready,
                                    // пока тот не станет равен 1.

                                    // После чего запускаем коллайдер, записав
    collider_ptr->start = 1;        // 1 в контрольный регистр start
  }
}

#define DEADLY_SERIOUS_EVENT 0xDEADDAD1

// extern "C" нужно использовать только в С++. Благодаря этому, в объектном
// файле функция будет называться именно int_handler, как и ожидает компоновщик
// при объединении кода с startup.S
// Без extern "C", при компиляции C++ кода имя функции в объектном файле будет
// немного другим (что-то типа _Z11int_handlerv), из-за чего возникнут проблемы
// в компоновке.
extern "C" void int_handler()
{
  // Если от коллайдера приходит прерывание, сразу же проверяем регистр статуса
  // и если его код равен DEADLY_SERIOUS_EVENT, экстренно останавливаем коллайдер
  if(DEADLY_SERIOUS_EVENT == collider_ptr->status)
  {
    collider_ptr->emergency_switch = 1;
  }
}
```

---

### Порядок выполнения задания

1. Написать программу для своего индивидуального задания на языке C или C++.
2. [Скомпилировать](#практика) программу и [стартап-файл](startup.S) в объектные файлы.
3. Скомпоновать объектные файлы исполняемый файл, передав компоновщику соответствующий [скрипт](linker_script.ld).
4. Экспортировать из объектного файла секции `.text` и `.data` в текстовые файлы `init_instr.mem`, `init_data.mem`. Если вы не создавали инициализированных статических массивов или глобальных переменных, то файл `init_data.mem` может быть оказаться пустым.
   1. Если файл `init_data.mem` не пустой, необходимо проинициализировать память в модуле `ext_mem` c помощью системной функции `$readmemh` как это было сделано для памяти инструкций.
   2. Перед этим из файла `init_data.mem` необходимо удалить первую строку (вида `@00001000`), указывающую начальный адрес инициализации.
5. Добавить получившиеся текстовые файлы в проект Vivado.
6. Запустить моделирование исполнения программы вашим процессором. Для отладки во время моделирования будет удобно использовать дизасемблерный файл, ориентируясь на сигналы адреса и данных шины инструкций.
7. Проверить исполнение программы процессором в ПЛИС.

---
