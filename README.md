Оригинальная статья [Parallelism, Concurrency, and AsyncIO in Python by example](https://testdriven.io/blog/python-concurrency-parallelism/)

В этом руководстве рассматривается, как ускорить операции, связанные с процессором и вводом-выводом, с помощью 
многопроцессорной обработки, многопоточности и AsyncIO.

# Concurrency vs Parallelism

**Concurrency** и **Parallelism** — похожие термины, но это не одно и то же.

**Concurrency** -- это возможность одновременного запуска нескольких задач на ЦП. Задачи могут запускаться, выполняться и 
завершаться в перекрывающиеся периоды времени. В случае одного процессора несколько задач запускаются с помощью 
переключения контекста, при этом состояние процесса сохраняется, чтобы его можно было вызвать и выполнить позже.

Между тем, **Parallelism** — это возможность одновременного запуска нескольких задач на нескольких ядрах ЦП.

Хотя они могут увеличить скорость вашего приложения, конкуренция и параллелизм не должны использоваться повсеместно. 
Вариант использования зависит от того, привязана ли задача к процессору или к вводу-выводу.

Задачи, которые ограничены ЦП, привязаны к ЦП. Например, математические вычисления связаны с процессором, поскольку 
вычислительная мощность увеличивается по мере увеличения количества компьютерных процессоров. Параллелизм предназначен 
для задач, связанных с процессором. Теоретически, если задача разделена на n-подзадач, каждая из этих n-задач может 
выполняться параллельно, чтобы эффективно сократить время исходной непараллельной задачи до 1/n. Параллелизм 
предпочтительнее для задач, связанных с вводом-выводом, так как вы можете делать что-то еще, пока ресурсы ввода-вывода 
извлекаются.

Лучший пример задач, связанных с процессором, — наука о данных. Специалисты по данным имеют дело с огромными массивами 
данных. Для предварительной обработки данных они могут разделить данные на несколько пакетов и запускать их параллельно, 
эффективно сокращая общее время обработки. Увеличение количества ядер приводит к более быстрой обработке.

Веб-скрапинг связан с вводом-выводом. Потому что задача мало влияет на ЦП, так как большая часть времени тратится на 
чтение и запись в сеть. Другие распространенные задачи, связанные с вводом-выводом, включают вызовы базы данных, а также 
чтение и запись файлов на диск. Веб-приложения, такие как Django и Flask, являются приложениями, связанными с 
вводом-выводом.

> Если вам интересно узнать больше о различиях между потоками, многопроцессорностью и асинхронностью в Python, 
> ознакомьтесь со статьей [Ускорение Python с помощью параллелизма, параллелизма и асинхронности](https://testdriven.io/blog/concurrency-parallelism-asyncio/).

# Сценарий

При этом давайте посмотрим, как ускорить следующие задачи:

```python
# tasks.py

import os
from multiprocessing import current_process
from threading import current_thread

import requests as requests


def make_request(num):
    # io-bound

    pid = os.getpid()
    thread_name = current_thread().name
    process_name = current_process().name
    print(f"{pid} - {process_name} - {thread_name}")

    requests.get("https://httpbin.org/ip")


async def make_request_async(num, client):
    # io-bound

    pid = os.getpid()
    thread_name = current_thread().name
    process_name = current_process().name
    print(f"{pid} - {process_name} - {thread_name}")

    await client.get("https://httpbin.org/ip")


def get_prime_numbers(num):
    # cpu-bound

    pid = os.getpid()
    thread_name = current_thread().name
    process_name = current_process().name
    print(f"{pid} - {process_name} - {thread_name}")

    numbers = []
    prime = [True for i in range(num + 1)]
    p = 2

    while p * p <= num:
        if prime[p]:
            for i in range(p * 2, num + 1, p):
                prime[i] = False
        p += 1

    prime[0] = False
    prime[1] = False

    for p in range(num + 1):
        if prime[p]:
            numbers.append(p)

    return numbers
```

> **_Заметки_**:
> * `make_request` отправляет HTTP-запрос на https://httpbin.org/ip X раз
> * `make_request_async` делает тот же HTTP-запрос асинхронно с [HTTPX](https://www.python-httpx.org/).
> * `get_prime_numbers` вычисляет простые числа с помощью метода [решета Эратосфена](https://en.wikipedia.org/wiki/Sieve_of_Eratosthenes) от двух до указанного предела.

Мы будем использовать следующие библиотеки из стандартной библиотеки, чтобы ускорить выполнение вышеуказанных задач:

* **threading** -- для одновременного выполнения задач
* **multiprocessing** -- для параллельного выполнения задач
* **concurent.future** -- для одновременного и параллельного запуска задач и единого интерфейса
* **ayncio** -- для параллельного исполнения задач с сопрограммами, управляемыми интерпретатором Python

| Library            | Class/Method        | Processing Type             |
|--------------------|---------------------|-----------------------------|
| threading          | Thread              | concurrent                  |
| concurrent.futures | ThreadPoolExecutor  | concurrent                  |
| asyncio            | gather              | concurrent (via coroutines) |
| multiprocessing    | Pool                | parallel                    |
| concurrent.futures | ProcessPoolExecutor | parallel                    |

# Операция, связанная с вводом-выводом

Опять же, задачи, связанные с вводом-выводом, тратят больше времени на ввод-вывод, чем на ЦП.

Поскольку веб-скрапинг связан с вводом-выводом, мы должны использовать многопоточность для ускорения обработки, 
поскольку извлечение HTML (IO) медленнее, чем его анализ (CPU).

Сценарий: как ускорить сценарий парсинга и сканирования веб-страниц на основе Python?

## Пример синхронизации

Начнём с эталона.

```python
# io-bound_sync.py

import time

from tasks import make_request


def main():
    for num in range(1, 101):
        make_request(num)


if __name__ == '__main__':
    start_time = time.perf_counter()

    main()

    end_time = time.perf_counter()
    print(f"Elapsed run time: {end_time - start_time} seconds.")
```

Здесь мы сделали 100 HTTP-запросов с помощью функции `make_request`. Поскольку запросы происходят синхронно, каждая 
задача выполняется последовательно.

```shell
Elapsed run time: 78.33852570499994 seconds.
```

Итак, это примерно 0,78 секунды на запрос.

## Пример потоковой передачи

```python
# io-bound_concurrent_1.py

import threading
import time

from tasks import make_request


def main():
    tasks = []

    for num in range(1, 101):
        tasks.append(threading.Thread(target=make_request, args=(num,)))
        tasks[-1].start()

    for task in tasks:
        task.join()


if __name__ == '__main__':
    start_time = time.perf_counter()

    main()

    end_time = time.perf_counter()
    print(f"Elapsed run time: {end_time - start_time} seconds.")
```

Здесь одна и та же `make_request` функция вызывается 100 раз. На этот раз используется библиотека `threading` для 
создания потока для каждого запроса.

```shell
Elapsed run time: 6.93611489199975 seconds.
```

Общее время уменьшается с ~78 секунд до ~7 секунд.

Поскольку мы используем отдельные потоки для каждого запроса, вам может быть интересно, почему все это не заняло ~0,78 с. 
Это дополнительное время является накладными расходами на управление потоками. Глобальная блокировка интерпретатора 
(GIL) в Python гарантирует, что только один поток одновременно использует байт-код Python.

## Пример concurrent.futures 

```python
# io-bound_concurrent_2.py

import time
from concurrent.futures import ThreadPoolExecutor, wait
from tasks import make_request


def main():
    futures = []

    with ThreadPoolExecutor() as executor:
        for num in range(1, 101):
            futures.append(executor.submit(make_request, num))

    wait(futures)


if __name__ == '__main__':
    start_time = time.perf_counter()

    main()

    end_time = time.perf_counter()
    print(f"Elapsed run time: {end_time - start_time} seconds.")
```

Здесь мы использовали `concurrent.futures.ThreadPoolExecutor` для достижения многопоточности. После того, как все 
фьючерсы/промисы созданы, мы ожидаем `wait`, пока все они будут завершены.

```shell
Elapsed run time: 18.193057960999795 seconds.
```

`concurrent.futures.ThreadPoolExecutor` на самом деле является абстракцией вокруг библиотеки `multithreading`, что 
упрощает ее использование. В предыдущем примере мы назначали каждый запрос потоку, и всего было использовано 100 потоков. 
Но `ThreadPoolExecutor` по умолчанию число рабочих потоков равно `min(32, os.cpu_count() + 4)`. `ThreadPoolExecutor` 
существует для облегчения процесса достижения многопоточности. Если вы хотите больше контролировать многопоточность, 
используйте `multithreading` библиотеку.

## Пример асинхронного ввода-вывода

```python
# io-bound_concurrent_3.py

import asyncio
import time
import httpx

from tasks import make_request_async


async def main():
    async with httpx.AsyncClient() as client:
        return await asyncio.gather(
            *[make_request_async(num, client) for num in range(1, 101)]
        )

if __name__ == '__main__':
    start_time = time.perf_counter()
    loop = asyncio.get_event_loop()
    loop.run_until_complete(main())

    end_time = time.perf_counter()
    elapsed_time = end_time - start_time
    print(f"Elapsed run time: {elapsed_time} seconds.")
```

```shell
Elapsed run time: 1.7367996760003734 seconds.
```

> Здесь используется `httpx` так, как `requests` не поддерживает асинхронные операции.

`asyncio` быстрее, чем другие методы, поскольку `threading` использует потоки операционной системы. Таким образом, 
потоки управляются ОС, где переключение потоков вытесняется ОС. `asyncio` использует сопрограммы, определенные 
интерпретатором Python. С сопрограммами программа решает, когда переключать задачи оптимальным образом. Это 
обрабатывается `even_loopin` в asyncio.

# Операции с привязкой к процессору

Сценарий: _Как ускорить простой скрипт обработки данных?_

## Пример синхронизации

Опять же, давайте начнем с эталона.

```python
# cpu-bound_sync.py

import time
from tasks import get_prime_numbers


def main():
    for num in range(1000, 16000):
        get_prime_numbers(num)


if __name__ == '__main__':
    start_time = time.perf_counter()
    main()
    end_time = time.perf_counter()
    print(f"Elapsed run time: {end_time - start_time} seconds.")
```

Здесь мы выполнили функцию `get_prime_numbers` для чисел от 1000 до 16000.

```shell
Elapsed run time: 45.81140053599847 seconds.
```

## Пример многопроцессорности

```python
# cpu-bound_parallel_1.py

import time
from multiprocessing import Pool, cpu_count

from tasks import get_prime_numbers


def main():
    with Pool(cpu_count() - 1) as p:
        p.starmap(get_prime_numbers, zip(range(1_000, 16_000)))
        p.close()
        p.join()


if __name__ == '__main__':
    start_time = time.perf_counter()
    main()
    end_time = time.perf_counter()
    print(f"Elapsed run time: {end_time - start_time} seconds.")
```

Здесь мы использовали `multiprocessing` для вычисления простых чисел.

```shell
Elapsed run time: 50.92833401300001 seconds.
```

## Пример `concurrent.future`

```python
# pu-bound_parallel_2.py

import time
from concurrent.futures import ProcessPoolExecutor, wait
from multiprocessing import cpu_count

from tasks import get_prime_numbers


def main():
    futures = []
    with ProcessPoolExecutor(cpu_count() - 1) as executor:
        for num in range(1_000, 16_000):
            futures.append(executor.submit(get_prime_numbers, num))

    wait(futures)


if __name__ == '__main__':
    start_time = time.perf_counter()
    main()
    end_time = time.perf_counter()
    print(f"Elapsed run time {end_time - start_time} seconds.")
```

Здесь мы достигли многопроцессорности, используя `concurrent.futures.ProcessPoolExecutor`. Как только задания добавлены 
в фьючерсы, `wait(futures)` ждет их завершения.


```shell
Elapsed run time 64.46625369200001 seconds.
```

`concurrent.futures.ProcessPoolExecutor` является оберткой вокруг `multiprocessing.Pool`. Он имеет те же ограничения, 
что и `ThreadPoolExecutor`. Если вы хотите больше контролировать многопроцессорность, используйте 
`multiprocessing.Pool.concurrent.futures` обеспечивает абстракцию как многопроцессорности, так и многопоточности, что 
позволяет легко переключаться между ними.

# Вывод

Стоит отметить, что использование многопроцессорности для выполнения `make_request` функции будет намного медленнее, 
чем многопоточность, поскольку процессам придется ждать ввода-вывода. Однако подход с многопроцессорной обработкой 
будет быстрее, чем подход с синхронизацией.

Точно так же использование параллелизма для задач, связанных с ЦП, не стоит усилий по сравнению с параллелизмом.

При этом использование параллелизма или параллелизма для выполнения ваших скриптов добавляет сложности. Ваш код, как 
правило, будет сложнее читать, тестировать и отлаживать, поэтому используйте их только в случае крайней необходимости 
для долго выполняющихся сценариев.

`concurrent.futures` это с чего я обычно начинаю, так как-

1) Легко переключаться между параллелизмом и параллелизмом.
2) Зависимым библиотекам не нужно поддерживать `asyncio` (`requests` vs `httpx`).
3) Это чище и легче читать по сравнению с другими подходами.


# Результаты выполнения AMD Ryzen 5 4600U

```bash
sudo lshw -C CPU
  *-cpu                     
       описание: ЦПУ
       продукт: AMD Ryzen 5 4600U with Radeon Graphics
       производитель: Advanced Micro Devices [AMD]
       физический ID: 6
       сведения о шине: cpu@0
       версия: 23.96.1
       серийный №: Null
       слот: FP6
       размер: 1429MHz
       capacity: 4GHz
       разрядность: 64 bits
       частота: 100MHz
       возможности: lm fpu fpu_exception wp vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht syscall nx mmxext fxsr_opt pdpe1gb rdtscp x86-64 constant_tsc rep_good nopl nonstop_tsc cpuid extd_apicid aperfmperf rapl pni pclmulqdq monitor ssse3 fma cx16 sse4_1 sse4_2 movbe popcnt aes xsave avx f16c rdrand lahf_lm cmp_legacy svm extapic cr8_legacy abm sse4a misalignsse 3dnowprefetch osvw ibs skinit wdt tce topoext perfctr_core perfctr_nb bpext perfctr_llc mwaitx cpb cat_l3 cdp_l3 hw_pstate ssbd mba ibrs ibpb stibp vmmcall fsgsbase bmi1 avx2 smep bmi2 cqm rdt_a rdseed adx smap clflushopt clwb sha_ni xsaveopt xsavec xgetbv1 xsaves cqm_llc cqm_occup_llc cqm_mbm_total cqm_mbm_local clzero irperf xsaveerptr rdpru wbnoinvd cppc arat npt lbrv svm_lock nrip_save tsc_scale vmcb_clean flushbyasid decodeassists pausefilter pfthreshold avic v_vmsave_vmload vgif v_spec_ctrl umip rdpid overflow_recov succor smca cpufreq
       конфигурация: cores=6 enabledcores=6 microcode=140509446 threads=12

```


| file 				| Result					|
|-------------------------------|-----------------------------------------------|
| cpu-bound_sync.py 		| Elapsed run time: 13.37443888200005 seconds.	|
| cpu-bound_parallel_1.py 	| Elapsed run time: 3.0153799019999497 seconds.	|
| cpu-bound_parallel_2.py 	| Elapsed run time 4.327618240999982 seconds.	|
| io-bound-sync.py		| Elapsed run time: 120.2351885600001 seconds.	|
| io-bound_concurrent_1.py	| Elapsed run time: 23.458888632999788 seconds.	|
| io-bound_concurrent_2.py	| Elapsed run time: 25.870434013999784 seconds. |
| io-bound_concurrent_3.py	| Elapsed run time: 1.8479552819999299 seconds.	|


