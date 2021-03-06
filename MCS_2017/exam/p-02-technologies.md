## Технологии разработки параллельных программ. Параллельные библиотеки, актуальные для предметной области. Параллельные библиотеки общего назначения. Параллельные языки программирования. Модели параллельных программ, которые задаются языками и библиотеками. DSL — языки параллельного программирования.

Процесс — компьютерная программа в стадии своего выполнения. Компьютерная программа является пассивным набором инструкций; процесс является активным выполнением этих инструкций.

Процессор, по сути, представляет собой схему из функциональных элементов, причем результат выполнения схемы выдается только через некоторый промежуток времени. В связи с этим возникают следующие понятия:

* такт процессора — минимальное время, за которое можно гарантировано прочитать схему в установившемся положении;

* тактовая частота процессора — число тактов процессора, совершаемых им в единицу времени.

Можно построить пирамиду технологий по принципу удаленности от железа.

* Первый уровень — уровень ОС. Это то, что нам предоставляет операционная система. Примеры: pthread, socket, fork, clone, то есть набор системных вызовов. Глубже программист уже не имеет контроля (за исключением векторных инструкций, которые мы рассматривать не будем). Цель уровня: обеспечивать переносимый относительно конретного железа способ получения параллельных ресурсов. А именно: программисту все равно, работает он на процессоре Intel или нет, с этим будет разбираться операционная система, системные же вызовы будут одними и теми же для любого железа.

* Второй уровень — низкоуровневые библиотеки параллельного программирования. Стоит отметить, что с точки зрения ОС, это библиотеки высокого уровня, т.к. это не системные вызовы а некоторые обертки над ними, использующие стандартную библиотеку языка и т. д. Примеры: MPI, CUDA, OpenMP, OpenShmem. Задача этого уровня: абстрагироваться от операционной системы и от конкретной идеологии передачи данных. Разберем это на примерах:

  * В MPI есть некоторый интерфейс в виде процессов, одновременно выполняющих один код. При этом передача данных между процессами может происходить с помощью технологии InfiniBand (если процессы на разных машинах), а может происходить с помощью технологии IPC Shared Memory (если на одной машине) и программисту абсолютно не важно, как именно это будет происходить: интерфейс у него одинаковый для 2 случаев. Если программист пишет в своем коде `MPI_Send`, то сама RunTime-библиотека MPI в процессе исполнения выбирает оптимальный в данной ситуации механизм передачи данных и либо инициирует передачу по сети, либо создает объект разделяемой памяти. Кроме того, если в программе используется ввод-вывод, то MPI может понять, что нужно использовать некоторое сетевое хранилище (storage) для этого, и сделать соответствующий запрос к сети Infiniband, программисту же ничего дополнительно делать не нужно. Коллективные операции также могут поддерживаться с использованием разных механизмов. И да, MPI обеспечивает переносимый относительно ОС интерфейс, но это скорее дополнительный бонус, а не главное преимущество.

  * OpenMP также предоставляет независимый интерфейс, который, однако, должен поддерживаться компилятором. Мы получаем независимость от ОС, причем с гарантией того, что способ распараллеливания будет выбран наиболее оптимально. Например, в Linux будет однозначно выбран pthread как наиболее оптимальный способ распараллеливания, а в Windows pthread выбран не будет. В Android будет выбран java-thread или еще что-нибудь, что хорошо работает на Android.

  * CUDA обеспечивает переносимый способ передачи чего-либо на графическую карту и запуска чего-либо на графической карте. Причем переносимый как с точки зрения ОС (работает под Windows, Linux и Mac), так и с точки зрения компилятора (компилятор nvcc собирает код, который будет выполняться на GPU, в объектный файл, а его уже можно линковать как угодно). Работает только с видеокартами от Nvidia.

  * OpenCL — имеет назначение, аналогичное CUDA, но является открытым стандартом (разработку поддерживал во многом AMD). Переносим (при этом будет выполняться на всех видеокартах, реализовавших стандарт), будет эмулироваться на процессоре, если подходящего графического устройства не нашлось, но умеет мало. Считается, что API у NVIDIA CUDA более продуман.

  * OpenSHMEM — PGAS (Partitioned Global Address Space)-библиотека, предоставляющая односторонние коммуникации, атомарные и коллективные операции. Если сеть поддерживает технологию RDMA (Remote Direct Memory Access) (например, в сети Infiniband), то прямой доступ к чужой памяти осуществляется без вовлечения целевого узла. Если сеть не поддерживает этой технологии, то библиотека работает так же, как MPI. Возможности при этом в этой библиотеке более узкие, чем у MPI. В частности, абсолютной переносимости между архитектурами эта библиотека, в отличие от MPI, не предоставляет: нельзя запустить программу на кла- стере, на котором половина машин работает на Linux, а половина машин работает на Windows (удивительно, но на MPI это можно сделать).

* Третий уровень

  * Обертки над технологиями второго уровня. Обертки могут выглядеть, как портирование библиотек на высокоуровневые языки. Пример: binding Python MPI. Вы пишете Python-программу, она использует C-библиотеку MPI, получая тем самым возможность сетевого взаимодействия из Python-программы. Есть такая же привязка Java к MPI, но это более экзотический пример.

  * Высокоуровневые библиотеки параллельного программирования. Это библиотеки, у которых в качестве низкого уровня используются MPI, OpenMP, CUDA, OpenShmen. Примеры таких библиотек:

    * BLAS (Basic Linear Algebra Subprograms) — общий интерфейс для библиотек по работе с линейной алгеброй. Реализации:
      * IBM: ESSL, PESSL
      * Intel: MKL
      * NVIDIA: Сublas

    * Curandom — библиотека для вычисления случайных чисел на графической карте (используется CUDA).

    * FFTW (Fastest Fourier Transform in the West) — библиотека, предоставляющая быстрое преобразование Фурье в MPI. Т.е. у вас запускается параллельная программа, считающая быстрое преобразование Фурье на вашем кластере, о чем сам программист может даже не подозревать.

    * Hypre — библиотека для работы с линейной алгеброй средствами MPI и OpenMP. У этой библиотеки есть предварительная стадия в виде настройки на архитектуру. Запускается набор тестов, определяющий, например, количество кешей и размеры кешей.

    * Plasma — библиотека для решения задач линейной алгебры на графических процессорах. Построена на концепции DataFlow: на видеокарте имеется множество элементов, каждый из которых может получить некоторую программу, после чего имеем дело с фронтом выполняющихся параллельно заданий.

    * PETSc — еще одна популярная библиотека по работе с линейной алгеброй. Поддерживает MPI и GPUs через CUDA или OpenCL.

  * Высокоуровневые языки параллельного программирования. Пишется язык, который будет компилироваться в код на MPI, OpenMP, CUDA или OpenSHMEM. Тем самым компиляция программ будет двухуровневой: сначала код на языке высокого уровня транслируется в низкоуровневый код(например C++ и MPI), который затем уже другим компилятором компилируется в машинный код. Примеры таких языков:

    * DVM — сильно расширенный OpenMP. Вы пишете программу на Fortran, делаете некоторые вставки (как pragma в OpenMP). После компиляции вы получаете совмещенную программу, которая работает на MPI и OpenMP.

    * Charm++ — язык программирования, программы на котором разбиваются на несколько взаимодействующих процессов (chare), которые умеют обмениваться друг с другом активными сообщениями. Активное сообщение — сообщение состоящее из данных и кода, который нужно выполнить для его обработки после приема. Вообще говоря, chare — С++ объект. Программа на charm++ ком- пилируется в C++. ООП модель этого языка позволяет избежать глобальных синхронизаций.

    Есть языки, которые поддерживают модель программирования PGAS (partitioned global address space, Разделенное глобальное адресное пространство). Примеры: X10, UPS. Cluster OpenMP тоже внутри устроен почти по технологии PGAS. Принцип модели: есть n процессов, у каждого своя память, внутри которой можно выделить некоторое окно, которое будет синхронизироваться для всех процессов. Вы пишете переменную в окно, делаете вызов, и переменная появляется в этом окне у каждой программы. Создается иллюзия общей памяти, в то время как на самом деле этой общей памяти нет. Такие языки хорошо работают на архитектуре NUMA, а также на кластерной архитектуре при наличии Remote Memory Access.

* Четвертый уровень — библиотеки и языки предметной области. Обычно используют внутри себя второй уровень, но могут ис-
пользовать и третий уровень.

  Здесь программист, написав что-то, не подозревает, что это будет исполняться параллельно и не подозревает, на каком конкретно языке это будет исполняться параллельно, уж тем более ему непонятно с использованием какой параллельной библиотеки это будет происходить. Ему предоставляется язык, понятный ему в терминах его предметной области. Такие языки называют DSL-языками (Domain Specific Language). Вы можете перемножать матрицы, передвигать атомы, а это все преобразуется в некоторую параллельную программу. Все детали того, как это работает, от вас скрыты.

  Примеры таких языков:

  * Норма — решение вычислительных задач сеточными методами. Позволяет описывать операции над сетками, в том числе действия, которые нужно выполнить в узлах сетки. Нужно для решения различных сеточных уравнений (теплопроводности, колебаний, Навье-Стокса). К параллельному коду программист доступа не имеет. Можно получить скомпилированную версию программы на MPI и попытаться ее редактировать, но это сравнимо с редактированием assembler-кода, полученного после компиляции программы на языке C++.

  * GAMESS — вычислительная квантовая химия.

  * Gromacs (CUDA), Namd (Charm++) — молекулярная динамика.

  * Fluent — универсальный программный комплекс, предназначенный для решения задач механики жидкостей и газов.

  * Flow Vision — трехмерные уравнения динамики жидкости и газа.
