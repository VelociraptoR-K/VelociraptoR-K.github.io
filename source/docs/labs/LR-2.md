# Лабораторная работа: Основы NumPy (отчёт не закончен)

## 📋 Цель работы

Изучить основные возможности библиотеки NumPy для численных вычислений и анализа данных: создание массивов, векторные и матричные операции, статистический анализ и визуализацию.

## 📝 Задание

1. Реализовать функции для создания и обработки массивов
2. Выполнить векторные и матричные операции
3. Провести статистический анализ данных
4. Визуализировать результаты
5. Обеспечить соответствие кода стандартам PEP-8, наличие аннотаций типов и документации

## 💻 Реализация

### Основные функции

#### Создание и обработка массивов
```python
def create_vector():
    """Создать массив от 0 до 9"""
    return np.arange(10)

def create_matrix():
    """Создать матрицу 5x5 со случайными числами"""
    return np.random.rand(5, 5)

def reshape_vector(vec):
    """Преобразовать (10,) -> (2,5)"""
    return vec.reshape(2, 5)

def transpose_matrix(mat):
    """Транспонирование матрицы"""
    return mat.T
```
Векторные операции
```python 
def vector_add(a, b):
    """Сложение векторов"""
    return a + b

def scalar_multiply(vec, scalar):
    """Умножение вектора на число"""
    return vec * scalar

def elementwise_multiply(a, b):
    """Поэлементное умножение"""
    return a * b

def dot_product(a, b):
    """Скалярное произведение"""
    return np.dot(a, b)
```
Матричные операции
```python 
def matrix_multiply(a, b):
    """Умножение матриц"""
    return a @ b

def matrix_determinant(a):
    """Определитель матрицы"""
    return np.linalg.det(a)

def matrix_inverse(a):
    """Обратная матрица"""
    return np.linalg.inv(a)

def solve_linear_system(a, b):
    """Решение системы Ax = b"""
    return np.linalg.solve(a, b)
```