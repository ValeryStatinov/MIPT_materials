# Какие этапы работы MapReduce-приложения должен реализовать разработчик?

Разработчику приложения для Hadoop _MapReduce_ необходимо реализовать базовый обработчик, который на каждом вычислительном узле кластера обеспечит преобразование исходных пар «ключ / значение» в промежуточный набор пар «ключ / значение» (класс, если речь не о Streaming, реализующий интерфейс Mapper), и обработчик, сводящий промежуточный набор пар в окончательный, сокращенный набор (класс, реализующий интерфейс _Reducer_).

Все остальные фазы выполняются программной моделью _MapReduce_ без дополнительного кодирования со стороны разработчика. Кроме того, среда выполнения Hadoop _MapReduce_ выполняет следующие функции:
* планирование заданий;
* распараллеливание заданий;
* перенос заданий к данным;
* синхронизация выполнения заданий;
* перехват «проваленных» заданий;
* обработка отказов выполнения заданий и перезапуск проваленных заданий;
* оптимизация сетевых взаимодействий.

_В случае Streaming_.
* _Map_:
  * определить входной формат;
  * обработка данных;
  * определить выходной формат.
* _Reduce_:
  * определить входной формат;
  * агрегирование данных, отсортированных по ключу;
  * обработка данных;
  * определить выходной формат.