# 设计模式

## 单例模式

```python
class Singleton:
    _instance = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
```

## 工厂模式

```python
class AnimalFactory:
    @staticmethod
    def create_animal(animal_type):
        if animal_type == "dog":
            return Dog()
        elif animal_type == "cat":
            return Cat()
```

## 观察者模式

```python
class Subject:
    def __init__(self):
        self.observers = []
    
    def notify(self, event):
        for observer in self.observers:
            observer.update(event)
```