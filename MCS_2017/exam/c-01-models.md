## Модель вычислений: процессоры и сеть, аналогия с разделяемой памятью. Cинхронные / асинхронные вычисления. Разделяемая память / message passing. Синхронайзер. Тезисно объяснить эквивалентность систем без отказов.

### Модели вычислений

Говорят, что лучшее определение распределенной системе дал Л. Лампорт:
> _Распределенная система_ — это система, которая может сделать ваш компьютер бесполезным, если откажет один из серверов, о существовании которого вы даже не подозревали.в

Оно описывает систему хорошо, но с ним тяжело работать. Придумаем какую-нибудь другую модель.

В системе должны быть как-то представлены работающие процессоры и средство коммуникации.

Существует две степени свободы:
* коммуникация: _shared memory_ (несколько процессоров взаимодействуют через общую память) vs. _message passing_ (интернет, где можно отправить или получить пакет);
* выполнение: _synchronous_ vs. _asynchronoous_ (описание того, как в целом работает система).

_Передача сообщений_:
* система состоит из вычислительных узлов, объединенных в сеть. На каждом вычислительном узле запущен экземпляр процесса (ровно один), и у него своя собственная память;
* вычислительная узлы не имеют общей памяти и взаимодействуют только через передачу сообщений.

_Общая память_:
* процессоры взаимодействуют через доступ к общей памяти и через нее как-то договариваются.

_Синхронная модель выполнения_:
* вычисления происходят тактами: 0, 1, 2 и т. д.;
* узлы имеют локальные монотонные часы, часы не согласованы (могут быть разные у разных процессоров), но есть ограничение на отклонение часов друг от друга;
* в message passing задержки передачи в сети конечные и укладываются в 1 логический такт (то есть если сообщение отправлено в одном такте, то другой поток обязательно его получит в этом же такте);
* в shared memory все обращения одним процессором к общей памяти в 1 логический такт (успеем произвесети всю запись, все чтение и т. п.);
* суть в том, что процессы что-то делают в рамках одного такта, потом ждут, когда такт закончится, и с началом нового продолжают дальше; а такты замеряет как бы кто-то глобальный.

_Асинхронная модель выполнения_:
* узлы имеют локальнаые монотонные часы, часы несогласованы, отклонение между часами неограничено 9;
* в message passing задержки передачи в сети конечные, но неограниченные (то есть сообщение придет, но неизвестно когда);
* в shared memory задержки доступа к общей памяти непредсказуемы, но конечные (та же ситуация).

### Эквивалентность различных моделей

Сокращения:
* SSM — Synchronous Shared Memory;
* ASM — Asynchronous Shared Memory;
* SMP — Synchronous Message Passing;
* AMP — Asynchronous Message Passing.

Таким образом, по двум степеням свободы получили четыре класса распределенных систем. Однако оказывается, что в некотором смысле системы эквиваленты: каждую модель можно эмулировать поверх другой. Правда, это верно только в случае отсутствия отказов.

* _ASM, SSM через AMP, SMP_:
  * будем использовать адресное пространство вычислительных узлов как общее адресное пространство;
  * операции `store` и `load` будем эмулировать через отправку отправку сообщений соответствующему узлу (посылаем нужному процессу, владельцу интересуюзей ячейки памяти, запрос «хочу прочитать» или «хочу записать»);
  * если ячейка и так принадлежит нам, то просто читаем и пишем.

* _AMP, SMP через ASM, SSM_:
  * разобьем общее адресное пространство на _n_ неперескающихся множеств, для каждого процесса свой участок;
  * заведем _n^2_ очередей, и если нужно отправить сообщение, то добавляем его в очередь, а если хотим получать, то это чтение из очереди;
  * очереди здесь в смысле SPSC (Single Producer, Single Consumer);
  * есть нюансы, например, в астнхронной модели сообщения могут меняться местами — тогда вместо очередей можно использовать множества и при чтении доставать из него какое-то случайное сообщение;
  * но в принципе, если можем обеспечить тот же порядок, в котором сообщения доставлялись, то система все равно будет асинхронной, просто мы получим соблюдение более жестких требований, которые так-то для эмуляции не нужны.

* _ASM, AMP через SSM, SMP_:
  * синхронная модель — частный случай асинхронной («сообщение дойдет за один такт» — частный случай «сообщение дойдет когда-нибудь»), так что эмуляция не требуется.

* _SSM, SMP через ASM, AMP_:
  * это сложнее, так как здесь время одного такта как бы неопределенное;
  * для SM-систем вможно пользоваться обычными барьерами (как раз то место, где все потоки одновременно закаончивают что-то делать);
  * для MP-систем нам потребуется другой специальный примитив, Synchronizier, который позволит эмулировать тактовость синхронной системы (разберем дальше).

_Замечания_.
* Синхронные вычисления на самом деле не требуют глобальных часов. Можно обойтись и локальными, но с огарниченной разницей по времени между каждой парой потоков и с ограниченным временем выполнения каждого события.
* Пример SSM-системы — обычная параллельная программа с барьерами, которые используются как счетчик тактов.
* SMP-системы может быть полезен при проектировании баз данных. Это сложно сделать в совсем синхронном режиме, но можно написать синхронайзер, и тогда получим синхронность. В синхронном случае проще писать логику программу.
* Если отказов нет, то все систему эквивалентны, а значит, в каждой из них работает тот же набор теорем, которые мы доказывали в курсе Concurrency, например, wait-free универсальная конструкция.

### Синхронайзеры

_Синхронайзер_ используется для эмуляции синхронной системы поверх асинхронной. С его помощью будем эмулировать методы `send`, `receive` и генерацию событий «начало нового такта».

_Детали алгоритма_:
* потребуем, чтобы в рамках одного эмулируемого такта каждый процесс отправлял каждому другому ровно одно сообщение (инача сложно разбираться);
* если процесс ничег не хочет отправлять, то отправляем пустое сообщение;
* если больше одного, то отправим один раз комбинацию исходных;
* так как это в рамках одного такта, эмулируемый алгоритм не должен сломаться;
* как работаем: сначала
* синхронайзер есть свой у каждого процесса;
* синхронайзер начинает работу в такте 0, узнает, какие сообщения нужно отправить, а потом ждем, когда остальные синхронизаторы отправят ему ровно одно сообщения;
* когда синхронайзер получает сообщение от всех соседей, он переходит в следующий такт.

_Корректность алгоритма_:
  * синхронайзер в такте _x_ может получать сообщения только от соседей, находящихся в тактах _x_ или _x + 1_;
  * действительно, он не могу получить сообщения от синхронайзера с меньшим тактом, так как иначе он сам бы не перешел в такт _x_;
  * синхронайзеров с тактом _x + 2_ и больше не может быть, так как им для перехода из _x + 1_ в _x + 2_ потребуется получить сообщение от всех синхронайзеров, в том числе и данного синхронайзера в такте _x_, что невозможно по уже доказанному;
  * таким образом, разница в тактах между любыми двумя синхронайзерами не превосходит один, и синхронайзер с большим тактом не может получить сообщение от синхронайзера с меньшим тактом.

Теперь квадратик эмуляции сошелся: по кругу можем эмулировать все 4 системы. Кстати, можно заметить, что _на асинхронных системах можно реализовывать более сложные вещи, чем на синхронных_ (так как можно сделать и синхронные, используя синхронизаторы). Это логично, так как у асинхронной системы нет глобального счетчика тактов, то есть мы свободнее.

Заметим, что если есть отказы и система асинхронная, то синхронизатор не получится сделать, то есть четыре системы не эквивалентны.
