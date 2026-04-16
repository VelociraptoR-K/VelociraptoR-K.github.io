# Классификация с применением Scikit-Learn

## Готовый ipynb-борд с заполненными ячейками по ссылке: ![ipynb](https://colab.research.google.com/drive/1tN9qmH7w6Mj3wk0s0PqvleIjevP2AxBP?usp=sharing)

### Пошаговый алгоритм интеграции обученной scikit-learn модели с веб-сервисом на Django:

**1\. Сохранение обученной модели**

*   Сериализовать модель с помощью `pickle` или `joblib`.
    
*   Пример: `joblib.dump(model, 'model.pkl')`.
    
*   Файл модели разместить в директории проекта (например, `ml_app/ml_models/`).
    

**2\. Создание Django-приложения**

*   Выполнить `python manage.py startapp prediction` для создания отдельного приложения.
    
*   Добавить приложение в `INSTALLED_APPS` в `settings.py`.
    

**3\. Загрузка модели при инициализации приложения**

*   В файле `prediction/apps.py` переопределить метод `ready()`.
    
*   Загрузить модель один раз при старте сервера и сохранить в переменную модуля или кэш.
    
*   Пример:

```python
from django.apps import AppConfig
import joblib
import os

class PredictionConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'prediction'
    model = None

    def ready(self):
        model_path = os.path.join(os.path.dirname(__file__), 'ml_models', 'model.pkl')
        PredictionConfig.model = joblib.load(model_path)
```

**4\. Создание формы для ввода данных**

*   В `prediction/forms.py` описать класс формы с полями, соответствующими признакам модели.
    
*   Указать типы полей (`FloatField`, `IntegerField`) и валидаторы.
    

**5\. Создание представления (view) для обработки запроса**

*   В `prediction/views.py` создать функцию-представление (или класс).
    
*   Для GET-запроса: отобразить пустую форму.
    
*   Для POST-запроса:
    
    *   Проверить валидность данных формы.
        
    *   Извлечь значения и преобразовать в массив признаков.
        
    *   Вызвать `PredictionConfig.model.predict()`.
        
    *   Вернуть результат в шаблон.
        

**6\. Настройка маршрутизации (URL)**

*   В `prediction/urls.py` определить путь к представлению (например, `predict/`).
    
*   Подключить URLs приложения в основном `urls.py` проекта через `include()`.
    

**7\. Создание шаблона (template)**

*   Создать HTML-шаблон с формой для ввода данных и блоком для отображения результата.
    
*   Использовать Django Template Language для подстановки предсказания.
    

**8\. Обработка CORS (при необходимости)**

*   Если планируется API-доступ из фронтенда на другом домене, установить и настроить `django-cors-headers`.
    

**9\. Тестирование и развертывание**

*   Запустить локальный сервер: `python manage.py runserver`.
    
*   Протестировать через браузер.
    
*   Для продакшена: настроить WSGI-сервер (Gunicorn) и веб-сервер (Nginx).

### Структура проекта (кратко):

project/  
├── manage.py  
├── project/  
│   ├── settings.py  
│   └── urls.py  
├── prediction/  
│   ├── apps.py           # загрузка модели в ready()  
│   ├── forms.py          # форма ввода  
│   ├── views.py          # логика предсказания  
│   ├── urls.py           # маршруты приложения  
│   ├── templates/  
│   │   └── prediction/  
│   │       └── predict.html  
│   └── ml\_models/  
│       └── model.pkl     # сохраненная модель