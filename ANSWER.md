**1)** Сигнал y1 реализовать проще, поскольку каждое значение y1[i] не зависит от y1[j] для других j, а значит это идеальная задача в
модели массового параллелизма (work item i будет вычислять значение y1[i]). Для сигнала y2[n] же необходимо будет писать reduction-like алгоритм, что сделать очевидно сложнее.

**2)** По условию в варпе будут потоки с одним и тем же ```get_local_id(1)```, поскольку размер варпа 32 и размер воркгруппы по оси X тоже 32.
```get_local_id(1) + get_local_size(1) * get_local_id(0)``` сравнимо с ```get_local_id(1)``` по модулю 32, т.к. ```get_local_size(1) = 32```. А значит ```idx % 32``` у каждого потока варпа будет одинаковым, следовательно все потоки будут исполнять только один бранч. То есть code divergence не будет.

**3)** 
(a)
Каждый поток в варпе сделает запись в одну и ту же кеш линию, поскольку индекс
```
get_local_id(0) + get_local_size(0) * get_local_id(1);
```
у соседних потоков в варпе будет отличаться на 1 и записи будут по адресам от X до X + 127, где X выровнен по 128 байтам.
Данное обращение к памяти безусловно coalesced. Поскольку варп будет накладываться 32 раза, будет 32 кеш линии записей.

(b)
Данное обращение к памяти точно не coalesced. Поскольку потоки i и i + 1 в варпе обратятся по индексам
``` a + 32 * i ``` и ``` a + 32 * (i + 1) ``` (a - константа), разница между ними будет равна 32, то есть 128 байт
или же размер 1 кеш линии. Значит потоки в варпе сделают 32 кеш линий записи.
При последовательном накладывании варпа потоки сделают 32 * 32 = 1024 кеш линий записи.

(c)
Данное обращение будет скорее coalesced, но не идеально из-за сдвига в 1.
Каждый поток с индексом от 0 до 30 сделает запись по адресам от X + 1 до X + 127, где X выровнен по 128 байтам.
Но поток с индексом 31 сделает запись по адресу X + 128 и из-за выравнивания придется загружать дополнительную кеш линию.
При последовательном накладывании варпа потоки сделают 2 * 32 = 64 кеш линий записи.