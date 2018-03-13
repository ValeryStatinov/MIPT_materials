## Модель OpenMP-программы. Переменные окружения используемые в OpenMP программах. Библиотечные функции. Сравнение с библиотекой pthread. Поддержка OpenMP со стороны компилятора, в том числе макросы, выставляемые для OpenMP-программ.

OpenMP – одно из наиболее популярных средств программирования для компьютеров с общей памятью, базирующихся на традиционных языках программирования и использовании специальных комментариев. OpenMP – один вариант программы для параллельного и последовательного выполнения. Разработчиком стандарта является некоммерческая организация OpenMP ARB (Architecture Review Board), в которую входят представители крупных компаний-разработчиков суперкомпьютеров и программного обеспечения. OpenMP поддерживает работу с языками Fortran и C/C++.

Прагма (из документации Microsoft) – «Директивы `#pragma` предоставляют каждому компилятору способ обеспечения специальных компьютерных функций и функций операционной системы при сохранении общей совместимости с языками C и C++». Основной особенностью прагм (директив) является то, что если компилятор не распознает данную прагму, то он должен ее игнорировать (в соответствие со стандартами ANSI C и C++).

В языке C директивы OpenMP оформляются указаниями препроцессору, начинающимиися с `#pragma omp`. Ключевое слов `omp` используется для того, чтобы исключить случайные совпадения имент директив OpenMP с другими именами в программе. Формат директивы на C/C++: `#pragma omp directive-name [option, ...]`. Все директивы OpenMP можно разделить на 3 категории: определение параллельной области, распределение работы, синхронизация.

Модель исполнения.
* OpenMP API использует fork-join модель параллельного исполнения. Потоки исполняют задачи, явно или неявно определнные директивами OpenMP.
* При старте OpenMP программы запускается один поток (initial thread). Он исполняется последовательно.
* Потоки объединены в группы (teams).
* При достижении директивы `parallel` порождается группа потоков, родительский поток также входит в эту группу потоков (master thread). С каждым потоком связывается задача, в которой выполняется код внутри директивы `parallel`. Задачи в разных потоках параллельно обрабатывают разные данные. После завершения всех потоков продолжает выполнение master thread. Для ожидания потоков используется барьер (неявно). Для того чтобы не ожидать завершения всех потоков, можно использовать директиву `nowait`. Возможна поддержка вложенных директив `parallel` (nested parallelism).
* Директива `task` позволяет создавать явные задачи, которые будут выполняться в одном из потоков текущей группы. Все задачи, созданные с помощью `task`, завершаются до завершения потоков данной группы.
* Также доступны и другие директивы, например, `cancel` (делает отмену самой внутренней охватывающей области указанного типа).

Модель памяти.
* Relaxed-Consistency модель пямти — значение локальных копий могут различаться в разных потоках.
* При создании потоков для переменной можно создать локальную копию для каждого потока (private). Private-перменные используются только в текущем потоке, могут инициализироваться значением оригинальной переменной, результат может быть сохранен в оригинальной переменной. Shared-переменные должны использоваться только на чтение либо защищаться с помощью директив синхронизации.
* Для синхронизации значения локальной копии переменной и переменной в памяти используется директива `flush`. Директива `flush` определяет место последнего изменения переменной и выполняет запись локального значения в память или чтение значения из памяти в локальную копию (при изменении значения в другом потоке). Программист должен обеспечить отсутствие конкурентной модификации переменной для которой выполняется `flush`.

Перменные окружения.
* `OMP_SCHEDULE`: `setenv OMP_SCHEDULE type[,chunk]`. Устанавливает тип распределения работ в параллельных циклах. Типы:
  * `static` — блочно-циклическое распределение итераций цикла; размер блока — `chunk`. Первый блок из `chunk` итераций выполняет нулевой поток, второй блок — следующий и т. д. до последнего потока, затем распределение снова начинается с нулевого потока. Если значение `chunk` не указано, то все множество итераций делится на непрерывные куски примерно одинакового размера, и полученные порции итераций распределяются мужду потоками;
  * `dynamic` — динамическое распределение итераций с фиксированным размером блока: сначала каждый поток получает `chunk` итераций, тот поток, который заканчивает выполнение своей порции итераций, получает первую свободную порцию из `chunk` итераций. Освободившиеся потоки получают новые порции итераций до тех пор, пока все порции не будут исчерпаны;
  * `guided` — динамическое распределение итераций, при котором размер порции уменьшается с некоторого начального значения до велечины `chunk` пропорционально количеству еще не распределенных итераций, деленному на количество потоков, выполняющих цикл;
  * `auto` — способ распределения итераций выбирается компилятором и/или системой выполнения.я
* `OMP_NUM_THREADS`: `setenv OMP_NUM_THREADS list`. Задает количество потоков, выполняющих параллельные области программы.
* `OMP_DYNAMIC`: `setenv OMP_DYNAMIC dynamic`. Если `true`, то разрешает, а иначе запрещает динамически изменять количество потоков, используемых для выполнения параллельной области.
* `OMP_NESTED`: `OMP_NESTED nested`. Если `true`, то разрешает, а иначе запрещает вложенный параллелизм.
* `OMP_STACKSIZE`: `setenv OMP_STACKSIZE size[B|K|M|G]`. Задает размер стека для создаваемых из программы потоков, буквы — единицы измерения (байты, килобайты, мегабайты и гигабайты).
* `OMP_WAITPOLICY`: `setenv OMP_WAITPOLICY policy`. Задает поведение ждущих процессов. Если задано значение `ACTIVE`, то ждущему процессу будут выдаваться циклы процессорного времени, а при значении `PASSIVE` ждущий процесс может быть отправлен в сппящий режим, при этом процессор может быть назначен дургим процессам.
* `OMP_MAX_ACTIVE_LEVELS`: `setenv OMP_MAX_ACTIVE_LEVELS levels`. Задает максимально допустимое количество вложенных параллельных областей.
* `OMP_THREAD_LIMIT`: `setenv OMP_THREAD_LIMIT limit`.Задает максимальное число потоков, допустимых в программе.
* И др.

Библиотечные функции.

* `void omp_set_num_threads(int num_threads)`. Устанавливает количество потоков, которое может быть запрошено для параллельного блока.
* `int omp_get_num_threads()`. Возвращает количество потоков в текущей группе потоков.
* `int omp_get_max_threads()`. Возвращает максимальное количество потоков, которое может быть установлено `omp_set_num_threads`.
* `int omp_get_thread_num()`. Возвращает номер потока в группе.
* `int omp_get_num_procs()`. Возвращает количество физических процессоров, доступных программе.
* `int omp_in_parallel()`. Возвращает ненулевое значение, если вызвана внутри параллельного блока. В противном случае возвращается `0`.
* `void omp_set_dynamic(int dynamic_threads)`. Разрешает или запрещает динамическое выделение потоков.
* `int omp_get_dynamic()`. Возвращает? разрешено или запрещено динамическое выделение потоков.
* `void omp_set_nested(int nested)`. Разрешает или запрещает вложенный параллелизм.
* `int omp_get_nested()`. Возвращает, разрешен или запрещен вложенный параллелизм.
* И др.

OpenMP поддерживается многими современными компиляторами:
* компиляторы Sun Studio
* Visual C++;
* GCC;
* Intel C++ Compiler;
* Clang / LLVM;
* и др.

Компилятор с поддержкой OpenMP определяет макрос `_OPENMP`, который может использоваться для условной компиляции отдельных блоков, характерных для параллельной версии программы. Этот макрос определен в формате `yyyymm`, где `yyyy` и `mm` — цифры года и месяца, когда был принят поддерживаемый стандарт OpenMP.

Сравнение с pthreads. POSIX-интерфейс для организации нитей (pthreads) поддерживается практически на всех UNIX-системах, однако по многим причинам не подходит для практического параллельного программирования: в нем нет поддержки языка Fortran, слишком низкий уровень программирования, нет поддержки параллелизма по данным, а сам механизм нитей изначально разрабатывался не для целей организации параллелизма. OpenMP можно рассматривать как высокоуровневую надстройку над pthreads (или аналогичными библиотеками нитей); в OpenMP используется терминология и модель программирования, близкая к pthreads, например, динамически порождаемые нити, общие и разделяемые данные, механизм «замков» для синхронизации.