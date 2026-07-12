# Django 框架概述

## 什么是 Django？

Django 是一个高级 Python Web 框架，鼓励快速开发和干净的设计。

## 核心特点

- **ORM**：强大的对象关系映射
- **Admin**：自动生成管理后台
- **安全**：内置安全防护
- **快速开发**：电池全包

## 快速开始

```bash
pip install django
django-admin startproject myproject
cd myproject
python manage.py runserver
```

## 模型

```python
from django.db import models

class User(models.Model):
    name = models.CharField(max_length=100)
    email = models.EmailField(unique=True)
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        ordering = ['-created_at']
    
    def __str__(self):
        return self.name
```

## 视图

```python
from django.shortcuts import render, get_object_or_404
from django.http import JsonResponse

def user_list(request):
    users = User.objects.all()
    return render(request, 'users/list.html', {'users': users})

def user_detail(request, pk):
    user = get_object_or_404(User, pk=pk)
    return JsonResponse({'name': user.name, 'email': user.email})
```

## REST Framework

```python
from rest_framework import serializers, viewsets

class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = '__all__'

class UserViewSet(viewsets.ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer
```

## URL 配置

```python
from django.urls import path
from . import views

urlpatterns = [
    path('users/', views.user_list),
    path('users/<int:pk>/', views.user_detail),
]
```

## Admin

```python
from django.contrib import admin
from .models import User

@admin.register(User)
class UserAdmin(admin.ModelAdmin):
    list_display = ('name', 'email', 'created_at')
    search_fields = ('name', 'email')
```
