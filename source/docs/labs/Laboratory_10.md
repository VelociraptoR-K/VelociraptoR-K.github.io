# Лабораторная работа №10

## Цель работы:
Исследовать методы оптимизации вычисления кода, используя потоки, процессы, 
Cython и отключение GIL на основе сравнения времени вычисления функции численного 
интегрирования методом прямоугольников, реализованной на чистом Python.

## Пример заданий:
1. Написать полный и содержательный docstring для функции integrate(), включая описание назначения каждого аргумента, возвращаемого значения и возможных ограничений метода. Формат docstring должен соответствовать стандартам PEP 257.
2. Использовать аннотации типов для аргументов и возвращаемых значений функции, применяя синтаксис типа подсказок согласно PEP 484. Убедитесь, что аннотации соответствуют типу передаваемых данных.
3. Создать два примера для проверки правильности работы функции, размещённые непосредственно в docstring функции integrate(). Один пример должен содержать простую тригонометрическую функцию, второй — полиномиальную функцию второго порядка.
4. Разработать дополнительные юнит-тесты, покрывающие хотя бы две ситуации: правильный расчёт известного интеграла и проверку устойчивости к изменению числа итераций. Для оценки точности используйте assertAlmostEqual или аналогичный механизм проверки близости чисел.
5. Провести оценку производительности функции с помощью модуля timeit, выполнив серию замеров времени выполнения функции для различного числа итераций. Фиксировать полученные временные характеристики и включить их в итоговый отчет.

## Отрывок кода из лабораторной, демонстрирующий оптимизацию для интегрирования:
```python
import concurrent.futures as ftres
from functools import partial
from typing import Callable, Union
import math
import time
import timeit


def integrate(f: Callable[[float], float],
              a: Union[int, float],
              b: float,
              *,
              n_iter: int = 10000) -> float:
    """
    Вычисляет площадь под графиком функции численным интегрированием

    Недостаток: плохо работает для быстро меняющихся функций с малым количеством прямоугольников

    Args:
        f: Функция одного вещественного аргумента, интеграл которой вычисляется.
        a: Левая граница интервала интегрирования.
        b: Правая граница интервала интегрирования.
        n_iter: Количество прямоугольников для аппроксимации интеграла.

    Returns:
        Приближенное значение определенного интеграла.

    Examples:
        >>> import math
        >>> result = integrate(math.sin, 0, math.pi, n_iter=10000)
        >>> abs(result - 2.0) < 0.01
        True

        >>> result = integrate(lambda x: x**2, 0, 1, n_iter=10000)
        >>> abs(result - 1/3) < 0.001
        True
    """

    if a >= b:
        raise ValueError(f"a ({a}) должно быть меньше b ({b})")

    acc = 0.0
    step = (b - a) / n_iter
    f_local = f
    a_local = a
    for i in range(n_iter):
        acc += f_local(a_local + i * step) * step
    return acc

def integrate_async(f: Callable[[float], float],
                    a: Union[int, float],
                    b: float,
                    *,
                    n_jobs: int = 2,
                    n_iter: int = 1000) -> float:
    """Параллельное интегрирование с потоками."""

    executor = ftres.ThreadPoolExecutor(max_workers=n_jobs)
    spawn = partial(executor.submit, integrate, f, n_iter=n_iter // n_jobs)
    step = (b - a) / n_jobs

    fs = [spawn(a + i * step, a + (i + 1) * step) for i in range(n_jobs)]
    executor.shutdown(wait=False)
    return sum(f.result() for f in ftres.as_completed(fs))


def integrate_async_processes(f: Callable[[float], float],
                              a: Union[int, float],
                              b: float,
                              *,
                              n_jobs: int = 2,
                              n_iter: int = 1000) -> float:
    """Параллельное интегрирование с процессами."""

    executor = ftres.ProcessPoolExecutor(max_workers=n_jobs)
    spawn = partial(executor.submit, integrate, f, n_iter=n_iter // n_jobs)
    step = (b - a) / n_jobs

    fs = [spawn(a + i * step, a + (i + 1) * step) for i in range(n_jobs)]
    executor.shutdown(wait=False)
    return sum(f.result() for f in ftres.as_completed(fs))


def benchmark_iterations():
    """
    Проводит замер времени выполнения функции integrate для различного количества итераций.

    Returns:
        list: Список с результатами замеров времени
    """
    print("-" * 60)
    print("Замеры integrate() с разным числом итераций:")
    print("-" * 60)

    test_cases = [
        (1000, "1K итераций"),
        (10000, "10K итераций"),
        (100000, "100K итераций"),
        (1000000, "1M итераций"),
    ]
    results = []
    for n_iter, description in test_cases:
        time_taken = timeit.timeit(
            lambda: integrate(math.cos, 0, math.pi, n_iter=n_iter),
            number=10
        ) / 10

        results.append(f"{description}: {time_taken:.6f} секунд")
    return results


def benchmark_threads_processes():
    n_jobs_list = [2, 4,6,8]
    print()
    print("-" * 75)
    print("Сравнение потоков и процессов:")
    print("-" * 75)

    for n_jobs in n_jobs_list:
        start = time.perf_counter()
        result_threads = integrate_async(math.sin, 0, math.pi, n_jobs=n_jobs, n_iter=5000)
        time_threads = time.perf_counter() - start

        start = time.perf_counter()
        result_processes = integrate_async_processes(math.sin, 0, math.pi, n_jobs=n_jobs, n_iter=5000)
        time_processes = time.perf_counter() - start

        print(f"n_jobs={n_jobs}:")
        print(f"  Потоки: {time_threads:.4f} сек, результат={result_threads:.6f}")
        print(f"  Процессы: {time_processes:.4f} сек, результат={result_processes:.6f}")
        print()

if __name__ == "__main__":
    benchmark_iterations()
    benchmark_threads_processes()
```
## Выводы: Сравнение потоков и процессов:
- n_jobs=2: Потоки: 0.0032 сек, результат=2.000000 Процессы: 0.0809 сек, результат=2.000000

- n_jobs=4: Потоки: 0.0010 сек, результат=2.000000 Процессы: 0.0758 сек, результат=2.000000

- n_jobs=6: Потоки: 0.0008 сек, результат=2.000000 Процессы: 0.0853 сек, результат=2.000000

- n_jobs=8: Потоки: 0.0015 сек, результат=2.000000 Процессы: 0.0946 сек, результат=2.000000

При численном интегрировании примитивы синхронизации (мьютексы, семафоры) не нужны, потому что:
1. Данные разделены (data parallelism):
- Каждый поток работает со своим интервалом
- Нет общей изменяемой структуры данных
- Результаты объединяются только в конце
2. Нет состояния гонки (race condition):

- Каждый аккумулятор - локальная переменная
- Суммирование результатов происходит ПОСЛЕ вычислений
- Нет одновременной записи в одну переменную

### С полным содержанием Лабораторной работы 10 можно ознакомиться по [ссылке:](https://github.com/VelociraptoR-K/Repo_Py/tree/main/Lab_10)