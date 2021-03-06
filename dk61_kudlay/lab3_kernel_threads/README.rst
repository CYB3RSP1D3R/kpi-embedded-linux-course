=============================================
Лабораторная работа №3: Связный список и потоки ядра  
=============================================

Цели и задание
-----
Изученение механизма создания потоков и работы со связным списком в ядре Linux. Динамическая аллокация памяти в модуле.
Механизмы синхронизации в ядре. 

Задание на третью лабораторную:

- Написать модуль ядра, который:
  - содержит переменную
  - запускает M потоков на одновременное выполнение
  - каждый поток инкрементирует переменную N раз, кладет значение переменной в список и завершается
  - при выгрузке модуль выводит значение переменной и содержимое списка
  - использовать параметры модуля для задания инкремента N и количества потоков M
  - для переменной, списка, потоков использовать динамическую аллокацию. Переменную передавать в поток по ссылке аргументом

- Реализовать функции lock() и unlock() с использованием атомарных операций ядра;

- Защитить доступ к шареным элементам. 
- Продемонстрировать правильность работы на x86 и BBXM


Теоретическая база работы 
-----

Связанный список — это самая простая и одна из чаще всего используемых структур данных в ядре Linux. 
Она позволяет сохранять переменное число элементов, называемых узлами (nodes) списка, и манипулировать ими. 
В отличие от элементов статического массива, элементы связанного списка можно динамически создавать и помещать их 
в сам список. Это позволяет на этапе выполнения программы обрабатывать произвольное количество элементов, 
которые еще не существовали во время компиляции программы, и точное их число было неизвестно. 
Поскольку элементы создаются в разные участки времени выполнения программы, 
они необязательно должны занимать соседние участки памяти компьютера. По этой причине элементы нужно както связать друг с другом, 
т.е. в каждом элементе списка должен храниться указатель на следующий элемент. 
Во время добавления или удаления элементов списка нужно просто скорректировать указатель на следующий узел.

Реализацию связанных списков в ядре Linux это не что иное, как циклически связанные двунаправленные списки. 
При реализации связанных списков в специальной переменной сохраняется указатель на первый элемент списка, 
называемый головным (head), или головой списка. Это позволяет быстро получить доступ к началу списка. 
Для того чтобы преобразовать эту структуру в элемент связанного списка, 
чаще всего к ней добавляют указатели на следующий и предыдущий элементы. Однако в ядре Linux применен совершенно иной подход. 
Вместо того чтобы превращать структуру в элемент связанного списка, в Linux элемент связанного списка (точнее, структура, которая его описывает) 
**встраивается в саму структуру данных**!

К примеру, встроим в стуктуру даннных экземпляр структуры, которая описывает связный список в ядре:

      .. code-block:: C
      
    struct kern_list {
        struct list_head lhead;
        int counter_val;
        int thread_index;
    };

Как можно увидеть, в структуре присутствует элемент ``lhead`` типа ``list_head``. Именно через структуру ``list_head`` происходит
взаимодействие с функциями связного списка в ядре. Но для начала, список нужно проинициализировать. При создании отдельного экземпляра
структуры данных пользователя путём динамической инициализации, можно использовать:

      .. code-block:: C

        struct list_head head;
        ...
        LIST_HEAD(&head);
        
Также можно создать общий указатель на спсиок путём использования ``LIST_HEAD(&head)``, где ``head`` экземпляр структуры
``list_head``.

Для базовой работы со списком необходимо знать как добавить элемент в список, как его удалить и как обойти список. 

Для добавления стоит использовать функцию ``list_add(struct list_head *new, struct list_head *head)``, где ``new`` указатель
на переменную типа ``list_head`` в структуре будущего элемента списка, ``head`` - указатель на головной элемент, определенный 
во время инициализации списка.

Для удаления элементов из списка стоит использовать функцию ``list_del(struct list_head *entry)``, где ``entry`` указатель на 
переменную типа ``list_head`` в структуре удаляемого элемента.

Для обхода всех элементов списка используется макрос ``list_for_each()``, 
которому передаются два параметра — оба структуры типа ``list_head``. Первый параметр — это указатель, 
который используется для обращения к текущему элементу списка. Он является временной переменнойю. 
Второй параметр — это структура типа ``list_head``, соответствующая головному узлу списка, 
который необходимо обойти. 
На каждой итерации цикла в первом параметре будет находиться указатель на очередной элемент списка,
пока не будут перебраны все его элементы.
Чтобы получить указатель на структуру, содержащую структуру типа ``list_head``, мы должны воспользоваться макросом list_entry(). 
Примеры использования в функциях для удаления списка ``delete_list`` и и вывода ``print_list``.

      .. code-block:: C
      
        static void delete_list(struct list_head **cursor, struct list_head *head)
        {
            struct list_head *tmp;
            struct kern_list *pk_list;
            
            list_for_each_safe (&cursor, tmp,  head) {
                pk_list = list_entry(&cursor, typeof(*pk_list), lhead);

                list_del(*cursor);
                kfree(pk_list);
            }
        }

        static void print_list(struct list_head *cursor, struct list_head *head)
        {
            struct kern_list *pk_list;

            list_for_each (cursor, head) {
                pk_list = list_entry(cursor, typeof(*pk_list), lhead);
                printk(KERN_INFO "Counter %d ticked %d\n",
                    pk_list->thread_index, pk_list->counter_val);
            }
        }
 
Часто требуется выполнить в ядре некоторые операции в фоновом режиме. В ядре такая возможность реализована в виде потоков 
ядра (kernel thread) — обычных процессов, которые выполняются исключительно в пространстве ядра. 
Существенным отличием между потоками ядра и обычными процессами является то, что потоки в ядре не имеют своего 
адресного пространства. Эти потоки работают только в пространстве ядра, и их контекст не переключается в пространство 
пользователя. Тем не менее потоки ядра планируются и вытесняются так же, как и обычные процессы. 
Поток ядра может быть создан только другим потоком ядра. 
Интерфейс для порождения нового потока ядра из существующего описан в файле ``<linux/kthread.h>`` 
и приведен ниже.

      .. code-block:: C
      
        struct task_struct *kthread_create(int (*thread_func)(void *data), void *data, const char namefmt[], .)

Где ``thread_func`` указатель на функцию обработчкик потока, ``data`` указатель на аргумент потока, ``namefmt`` - имя потока.
После создания процесс находится в незапущенном состоянии (unrunnable state). 
Для его запуска нужно явным образом вызвать функцию ``wake_up_process()``. 
Чтобы создать процесс и сразу же запустить его на выполнение, можно воспользоваться единственной функцией kthread_run(), 
как показано ниже. 

      .. code-block:: C
      
        struct task_struct *kthread_run(int (*threadfn)(void *data), void *data, const char namefmt[], ...) 

Описанная выше процедура запуска процесса в ядре реализована в виде макроса, в котором вызываются обе функции,
``kthread_create()`` и ``wake_up_process()``.

Следующим важным элементом, который будет рассмотрен в рамках данной работы будет механизмы синхронизации в ядре.
Определения, необходимые для использования неделимых целочисленных операций, находятся в файле ``<asm/atomic.h>``. 
Для некоторых аппаратных платформ существуют дополнительные уникальные средства, 
но для всех аппаратных платформ существует минимальный набор операций, которые используются в ядре повсюду. 
При написании кода ядра необходимо убедиться в том, что соответствующие операции доступны и правильно реализованы для всех 
аппаратных платформ. Объявление переменных типа ``atomic_t`` выполняется как обычно. 
По мере необходимости можно установить начальное значение этой переменной с помощью ``ATOMIC_INIT()``

Однако из-за сложности опредленный операций, которые должны выполняться неделимо, атомарных операций может быть недостаточно.
Для этого используют более сложные механизмы. В ядре ``Linux`` используются так называемые спин-блокировки (spin lock). 
Это такой тип блокировки, которая может быть захвачена не более чем в одном потоке команд. 
Если в каком-то из потоков команд будет предпринята попытка захвата спин-блокировки, 
которая удерживается другим потоком, возникает состояние конфликта. 
При этом первый поток входит в цикл и начинает в нем постоянно проверять, не освободилась ли уже требуемая блокировка. 
Поскольку код при этом постоянно находится в цикле. 
Если блокировка свободна, поток может сразу же ее захватить и продолжить свое выполнение. 
Циклическая проверка предотвращает ситуацию, когда в критическом участке кода одновременно может находиться более 
одного потока команд. Одна и та же спин-блокировка может захватываться в разных местах кода ядра.
В результате всегда будет гарантирована защита и синхронизация при доступе, например, к какой-нибудь структуре данных. 

В ядре Linux также могут быть использованы семафоры (semaphore). Это блокировки, которые переводят процессы в состояние 
ожидания. При попытке захвата семафора, который уже захвачен другой задачей, 
текущая задача помещается в очередь ожидания и замораживается. 
После этого процессор переходит к выполнению другой задачи. 
Как только требуемый семафор освобождается, одна из задач, находящаяся в очереди ожидания, активизируется, 
после чего она может захватить требуемый семафор.

Есть определенный перечень особенностей использования семафоров:

- Поскольку задача, которая хочет захватить блокировку, переводится в состояние ожидания до момента освобождения блокировки,
  семафоры хорошо подходят для реализации блокировок, которые могут удерживаться в течение длительного периода времени. 
- С другой стороны, не рекомендуется использовать семафоры для реализации блокировок, которые удерживаются 
  в течение короткого периода времени. Дело в том, что задержки, связанные с переводом процессов в состояние ожидания, 
  обслуживания очереди ожидания и последующей активизации процесса, могут легко превысить время, 
  в течение которого удерживается сама блокировка. 
- Так как в случае возникновения конфликта при захвате блокировки поток 
  будет переведен в состояние ожидания, семафоры можно захватывать только в контексте процесса. 
  Дело в том, что в контексте прерывания потоки не обрабатываются планировщиком. 
- При удержании семафора процесс может переходить в состояние ожидания, хотя обычно так не делают. 
  Это не приведет к тупиковой ситуации, когда другой процесс попытается захватить ту же блокировку 
  (просто он переводится в состояние ожидания и не мешает выполняться первому процессу). 
- При захвате семафора нельзя удерживать спин-блокировку, поскольку процесс может переходить в состояние ожидания 
  до освобождения семафора, а при удержании спин-блокировки в состояние ожидания переходить нельзя.

Последним будет рассмотрен механизм под названием мьютекс (``mutex``). Мьютекс представляется в виде структуры mutex. Он ведет себя так же, как и семафор, счетчик использования которого равен 
единице, но имеет более простой интерфейс, более высокую эффективность и налагает дополнительные ограничения при 
использовании. 
Особености мьютексов наведены ниже:

- В один и тот же момент времени мьютекс может удерживать только одна задача. 
  Другими словами, счетчик использования мьютекса всегда равен единице. 
- Тот процесс, который захватил мьютекс, должен обязательно его освободить. 
  Другими словами, нельзя захватить мьютекс в одном контексте, а затем освободить его в другом. 
  Это означает, что мьютекс не пригоден для решения сложных задач синхронизации между ядром и пространством пользователя. 
  Тем не менее в большинстве случаев они аккуратно захватываются и освобождаются в одном и том же контексте.
- Повторные захваты и освобождения мьютексов не разрешаются. Это означает, что нельзя повторно захватить 
  тот же самый мьютекс и нельзя освободить уже освобожденный мьютекс. 
- Процесс не может завершить свою работу до тех пор, 
  пока он не освободит мьютекс. 
- Мьютекс нельзя захватить в обработчике прерываний или в его нижней половине. 

Выполнение  
-----
В директории ``src`` данной лабораторной работы находится исходный файл модуля ядра ``multhd_mod.c`` 
с результатом заданий в рамках данной работы. Проведём небольшой анализ исходного кода:

#. Проведена инициализация структуры пользовательского типа ``k_list``. В структуре присутствует переменная ``count_val``,
   которая будет иметь в себе результирующее значение вычисленное в потоке. ``thread_cnt`` имеет номер созданного потока. Однако,
   это значение будет верно, если потоки запускаються один за другим. В противном случае, номера потоков будут совпадать с теми, которые
   прервали друг друга. ``lhead`` типа ``list_head`` внедряет механизм связного списка в структура пользователя. ``LIST_HEAD`` 
   инициализирует указатель на список. 
   
   .. code-block:: C

      static struct k_list {

        struct list_head lhead;
        int count_val;
        int thread_cnt;

      };
      
      LIST_HEAD(head_list); 


#. Проведено создание и выделение памяти для массива указатель на структуры потоков типа ``task_struct``. Создание 
   потоков и их запуск осуществляется с помощью ``kthread_run``. Общий фрагмент кода наведен ниже:
      
      .. code-block:: C
      
        static struct task_struct **kthreads = NULL;
        ...
        static int __init init_mod(void)
        {
        ...
        kthreads = kmalloc(sizeof(*kthreads) * threads_num, GFP_KERNEL);

        if (kthreads == NULL) {
            error_handler(THREAD_ALLOC_ERROR);
            clean_module();
            return -ENOMEM;
        }       
        ...
        }


#. Само обьявление и определение функции потоков наведено ниже:        
   
    .. code-block:: C
    
        int thread_func(void *thread_index) 
        {
            lock(varlock);

            for(register int i = 0; i < inc_times; i++){
                global_inc++;
            }

            kern_list *ptr = kmalloc(sizeof(struct kern_list), GFP_KERNEL);
            
            if (ptr == NULL) {
                error_handler(LIST_ALLOC_ERROR);
                return -ENOMEM;
            }
            
            ptr->counter_val = global_inc;
            ptr->thread_index = *((int*) thread_index);

            plist = ptr->lhead;
            list_add(plist,&head);

            unlock(varlock);

            return 0;
        }

#. Особого внимания заслуживают реализация функций блокировки, которые в рамках данной работы требуют выполнения за счёт 
   атомарный операций в ``asm/atomic.h``. Изначально переменная блокировки инициализирована нулём. Первый поток, который начал
   выполнение быстрее остальных, с помощью  ``atomic_cmpxchg`` устанавливает в переменную блокировки значение ``1``, но возвращает 
   нулевое значение и выходит из цикла выполнения. Остальные потоки на этом этапе останавливаються и ждут освобождение блокировки 
   функией ``unlock``, которая возрващает нулевое значение в переменную блокировки. Процесс повторяется заново. Общий листинг наведен ниже: 
   
    .. code-block:: C
   
        static void lock(atomic_t *variable)
        {
            while(atomic_cmpxchg(variable, 0, 1));
        }

        static void unlock(atomic_t *variable)
        {
            atomic_set(variable, 0);
        }

 
          
Сборка модуля и тестирование 
-----          
Процесс сборки и запуска проекта следующий:

#. Для автоматизированной сборки используется Kbuild. С помощью команды ``make`` производиться сборка и компиляция 
   модуля. Для кросс-компиляции можно также указать архитектуру, компилятор и директорию исходников.  
#. Для добавления модуля в ядро нужно использовать ``sudo insmod threadsmod.ko``. Для передачи аргумента с количеством потоков и степени инкрементации глобальной переменной
   стоит также добавить к команде ``M=<numbers_of_threads> N=<numbers_of_increment_count>``.
#. Для просмотра логов ядра можно использовать ``dmesg -k | tail -20``.   
#. Для удаления модуля нужно использовать ``sudo rmmod threadsmod.ko``.
#. Для удаления резульатов сборки можно использовать ``make clean`` и ``make tidy``.

Анализ полученных результатов 
-----   
Было проведено тестирование модуля на архитектурах ``x86`` и ``ARMv7 SoC Zynq-7000``.
 
.. code-block:: C


    [ 5212.446006] Counter 8 ticked 8000 
    [ 5212.446006] Counter 7 ticked 7000 
    [ 5212.446007] Counter 6 ticked 6000 
    [ 5212.446007] Counter 5 ticked 5000 
    [ 5212.446008] Counter 4 ticked 4000 
    [ 5212.446008] Counter 3 ticked 3000 
    [ 5212.446009] Counter 2 ticked 2000 
    [ 5212.446009] Counter 1 ticked 1000
    [ 5212.446009] Module successfully uninstalled.

Благодаря использованию блокировок, результат является абсолютно правильным, из-за очередности выполнения потоков. 
Данное поведение модуля абсолютно идентичное как для ``x86`` так и для ``ARMv7``, которые в ``asm/atomic.h`` имеют одинаковые по смыслу 
операции, но реализованы на разных ассемблерах. Без доработки кода и указания дополнительных условий на выходе получаем одинаковую 
работу на двух платформах.
