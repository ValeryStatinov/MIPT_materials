## Модель односторонних коммуникаций в MPI. Поддержка односторонних коммуникаций со стороны аппаратуры.

Односторонние коммуникации в MPI — механизм для поддержки Remote Memory Access. Напомним, что это механизм, при котором один процесс имеет возможность читать данные из памяти другого процесса и писать в память другого процесса без непосредственного участия CPU другого процесса (только с помощью сетевого оборудования). Для поддержки односторонних коммуникаций в MPI используют механизм, который называют окнами.

Строго говоря окно — это некоторая абстракция, привязанная к участку памяти некоторого процесса. Этот участок памяти может быть считан другим процессом, который был зарегистрирован на это же окно, а так же по этому адресу может быть записана некоторая информация, также другим процессом, который зарегистрирован на это окно.

Важно понимать, что для того, чтобы запись или чтение из окна произошло, сам процесс, который владеет памятью этого окна, никаких вызовов производить не должен — все происходит, вообще говоря, без его ведома. В этом главное отличие односторонних коммуникаций от коммуникаций типа «точка-точка» и групповых операций. И, собственно, поэтому они называются односторонними.

Для работы с окнами, нужно в первую очередь произвести регистрацию окна. Нужно также понимать, что неверно представлять себе окна как аналог разделяемой памяти. По факту у нас нет никакого широковещания и нет никакой одновременной синхронизации. После регистрации окна можно выполнять только 2 операции: положить что-то в окно данному процессу и считать что-то из окна данного процесса. Заметим, что на другие процессы это никак не влияет, в частности это не влияет даже на процесс, который пишет что-то в окно другого процесса. Вообще говоря, память под окно у некоторых процессов может быть не выделена, а они на это окно при этом зарегистрированы: они просто будут обращаться к процессам, у которых память под это окно выделена, что-то им туда писать и что-то оттуда считывать.

Важно понимать, что так как коммуникации являются односторонними, то процессы общения происходят безо всякой синхронизации. Как уже упоминалось ранее, процесс, которому что-то пишут в память может об этом никак не догадываться. Читая что-то из памяти другого процесса, у нас опять же нет гарантий, что мы получим нужные нам данные. Односторонние коммуникации дают полную свободу от синхронизации, тем самым обеспечивая большой выигрыш по времени. Для того чтобы получить хоть какой-то аналог синхронизации, нужно пользоваться другими функциями сетевого взаимодействия.

Перейдем к прототипам основных функций работы с окнами.

* `int MPI_Win_create(void *base, MPI_Aint size, int disp_unit, MPI_Info info, MPI_Comm comm, MPI_Win *win)` — создает окно. Коллективный вызов должен быть вызван всеми процессами коммуникатора `comm`. Основные аргументы: `base` — буфер в оперативной памяти процесса, который этот процесс дает под окно, `size` — размер этого окна (некоторые процессы могут указать `0`, если не хотят ничего выделять), `win` — выходной параметр, само окно.

* `int MPI_Put(const void *origin_addr, int origin_count, MPI_Datatype origin_datatype, int target_rank, MPI_Aint target_disp, int target_count, MPI_Datatype target_datatype, MPI_Win win)` — положить процессу, зарегистрированному на окно `win`, с данным рангом `target_rank`, в его окно со смещением `target_disp`, мои данные по адресу `origin_addr` в количестве `origin_count`.

* `int MPI_Get(void *origin_addr, int origin_count, MPI_Datatype origin_datatype, int target_rank, MPI_Aint target_disp, int target_count, MPI_Datatype target_datatype, MPI_Win win)` — посмотреть в окно другому процессу с рангом `target_rank`, зарегистрированному на окно `win`. Взять `target_count` элементов из его окна по смещению `target_disp`, и скопировать к себе в локальную память (за которую отвечает `origin_addr`).

Как уже обсуждалось, просто `MPI_Put` и `MPI_Get` никакой синхронизации не дают, и за объективностью данных никто не следит. Обеспечить синхронизацию можно следующим образом.

* `int MPI_Barrier(MPI_Comm comm)` — казалось бы, однозначный способ синхронизации. Но им пользоваться в этой ситуации не стоит, да и однозначной гарантии синхронизации он не дает. Заметим, что `MPI_Put` не является блокирующим вызовом. То, что процессы прошли секцию `MPI_Barrier`, еще не говорит о том, что запись была совершена!

* `int MPI_Win_fence(int assert, MPI_Win win)` — специальная функция, которая гарантирует, что начатые на окне операции чтения и записи завершатся после возврата из этой функции и не будут производиться после возврата. Тип операций выбирается в аргументе `assert`. Вызов с определенным флагом `assert`, напротив, разрешает исполнение некоторых операций на окне.

  На деле `MPI_Win_fence` — это обобщение 4 функций: `MPI_Win_post`, `MPI_Win_wait`, `MPI_Win_start`, `MPI_Win_complete`. Для каждого окна определяется 2 типа эпох: эпоха изменений (exposure epoch), в которую к окну применимы только операции типа `MPI_Put`, и эпоха доступа (access epoch), в которую к окну применимы только операции типа `MPI_Get`. Вызовы post-wait начинают и заканчивают эпоху изменений, вызовы start-complete — начинают и заканчивают эпоху доступа.

Функции `MPI_RGet` и `MPI_RPut` являются неблокирующим аналогом функций `MPI_Get` и `MPI_Put`, которые действуют аналогично `MPI_Isend` и `MPI_Irecv`, только для случая односторонних коммуникаций. Синтаксис у них схож с синтаксисом вызова функций `MPI_Get` и `MPI_Put`, за тем исключением, что добавляется аргумент `request` (аналогично семантике вызова всех неблокирующих MPI-операций).

Хотя мы говорили ранее, что вызовы функций `MPI_Get` и `MPI_Put` не являются блокирующими, это на самом деле не совсем так.
  * Вызов функциии `MPI_Get` блокирует выполнение до тех пор, пока буфер `origin_addr` не будет заполнен требуемой информацией из окна.
  * `MPI_RGet` не блокирует выполнение, сразу возвращая управление и объект `request`, по которому позже можно будет сделать операцию `wait`, чтобы дождаться заполнения буфера.
  * Вызов `MPI_Put` блокирует выполнение до тех пор, пока информация из буфера `origin_addr` не будет скопирована на шину или еще куда-то (в зависимости от реализации MPI и устройства интерконнекта кластера).
  * Вызов `MPI_RPut` не делает этого, возвращая `request`, по которому можно дождаться момента, когда этот буфер снова можно будет использовать. Использование буфера до этого момента может привести к неизвестным заранее последствиям. Эти функции появились, начиная с MPI 3.0.

Поддержка односторонних коммуникаций со стороны аппаратуры заключается в регистрации окон. При регистрации происходит больше действий, чем может показаться. Сетевому оборудованию (в случае Infiniband этим процессом занимается HCA (Host Channel Adapter)) задается отображение из идентификатора окна на выделенный диапазон адресов в памяти для данной машины. Когда придет команда односторонней коммуникации, HCA через высокоскоростной PCI Express (Peripheral Component Interconnect) получит доступ к нужному участку памяти и считает из него нужный фрагмент или запишет что нибудь в нужный фрагмент. По факту, каждый процесс хранит смещение (offset) для своего окна в своей оперативной памяти. Это смещение привязывается к абстракции окна и к нему имеет доступ HCA. При регистрации окна нужно задать отображение на HCA-адаптерах всех процессов коммуникатора, по которому регистрируется окно.