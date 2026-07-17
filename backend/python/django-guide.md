     1|
     2|
     3|# Python Django 详细技术文档
     4|
     5|## 1. Django 框架概述与设计理念
     6|Django 是一个基于 Python 的高层次 Web 框架，遵循 **MVT（Model-View-Template）** 架构模式。其核心设计理念包括：
     7|- **Batteries-included**：内置 ORM、Admin、认证、会话、缓存、国际化等模块，开箱即用。
     8|- **DRY（Don't Repeat Yourself）**：通过通用视图、表单系统、模板继承减少重复代码。
     9|- **松耦合组件**：各层（URL路由、视图、模型、模板）独立可替换。
    10|- **安全优先**：默认防御 CSRF、XSS、SQL 注入、点击劫持等常见攻击。
    11|- **快速开发**：提供完善的开发工具链（`manage.py`、迁移系统、调试面板）。
    12|
    13|```bash
    14|# 创建项目
    15|django-admin startproject config .
    16|python manage.py startapp core
    17|```
    18|
    19|---
    20|
    21|## 2. 项目搭建与目录结构
    22|标准 Django 项目结构如下：
    23|```
    24|myproject/
    25|├── manage.py                 # 命令行工具入口
    26|├── config/                   # 项目配置包
    27|│   ├── __init__.py
    28|│   ├── settings.py           # 全局配置
    29|│   ├── urls.py               # 根路由
    30|│   ├── asgi.py / wsgi.py     # 部署接口
    31|├── core/                     # 业务应用
    32|│   ├── migrations/           # 数据库迁移文件
    33|│   ├── admin.py              # Admin 后台注册
    34|│   ├── apps.py               # 应用配置
    35|│   ├── models.py             # 数据模型
    36|│   ├── views.py              # 视图逻辑
    37|│   ├── urls.py               # 应用路由
    38|│   └── templates/            # 模板文件
    39|└── static/                   # 静态资源
    40|```
    41|
    42|---
    43|
    44|## 3. 模型层（Models）
    45|### 字段类型与关系
    46|```python
    47|# core/models.py
    48|from django.db import models
    49|
    50|class Category(models.Model):
    51|    name = models.CharField(max_length=100, unique=True)
    52|    slug = models.SlugField(unique=True)
    53|
    54|    class Meta:
    55|        verbose_name_plural = "categories"
    56|        ordering = ["name"]
    57|
    58|class Article(models.Model):
    59|    STATUS_CHOICES = (
    60|        ("draft", "草稿"),
    61|        ("published", "已发布"),
    62|    )
    63|    title = models.CharField("标题", max_length=200)
    64|    content = models.TextField("正文")
    65|    author = models.ForeignKey(
    66|        "auth.User", on_delete=models.CASCADE, related_name="articles"
    67|    )
    68|    category = models.ForeignKey(Category, on_delete=models.SET_NULL, null=True)
    69|    tags = models.ManyToManyField("Tag", blank=True)
    70|    status = models.CharField(max_length=10, choices=STATUS_CHOICES, default="draft")
    71|    created_at = models.DateTimeField(auto_now_add=True)
    72|    updated_at = models.DateTimeField(auto_now=True)
    73|
    74|class Tag(models.Model):
    75|    name = models.CharField(max_length=50, unique=True)
    76|
    77|    def __str__(self):
    78|        return self.name
    79|```
    80|
    81|### 迁移命令
    82|```bash
    83|python manage.py makemigrations core   # 生成迁移脚本
    84|python manage.py migrate               # 执行迁移
    85|python manage.py showmigrations        # 查看迁移状态
    86|```
    87|
    88|---
    89|
    90|## 4. 视图层（Views）
    91|### FBV vs CBV
    92|- **FBV（Function-Based View）**：逻辑直观，适合简单场景。
    93|- **CBV（Class-Based View）**：支持方法派生、Mixin 复用，适合复杂 CRUD。
    94|
    95|```python
    96|# views.py
    97|from django.http import JsonResponse
    98|from django.views import View
    99|from django.views.generic import ListView, DetailView
   100|from .models import Article
   101|
   102|# FBV
   103|def article_list_fbv(request):
   104|    articles = Article.objects.filter(status="published")
   105|    return render(request, "core/article_list.html", {"articles": articles})
   106|
   107|# CBV
   108|class ArticleListView(ListView):
   109|    model = Article
   110|    template_name = "core/article_list.html"
   111|    context_object_name = "articles"
   112|    paginate_by = 20
   113|
   114|    def get_queryset(self):
   115|        return super().get_queryset().filter(status="published")
   116|
   117|class ArticleDetailView(DetailView):
   118|    model = Article
   119|    template_name = "core/article_detail.html"
   120|    context_object_name = "article"
   121|```
   122|
   123|---
   124|
   125|## 5. 模板系统（Templates）
   126|### 模板继承
   127|```html
   128|<!-- base.html -->
   129|<!DOCTYPE html>
   130|<html lang="zh">
   131|<head>
   132|    <meta charset="UTF-8">
   133|    <title>{% block title %}My Site{% endblock %}</title>
   134|</head>
   135|<body>
   136|    {% include "partials/header.html" %}
   137|    <main>
   138|        {% block content %}{% endblock %}
   139|    </main>
   140|    {% include "partials/footer.html" %}
   141|</body>
   142|</html>
   143|```
   144|
   145|```html
   146|<!-- article_list.html -->
   147|{% extends "base.html" %}
   148|{% block title %}文章列表{% endblock %}
   149|{% block content %}
   150|  <ul>
   151|    {% for article in articles %}
   152|      <li><a href="{% url 'article_detail' article.pk %}">{{ article.title|truncatewords:10 }}</a></li>
   153|    {% empty %}
   154|      <p>暂无文章</p>
   155|    {% endfor %}
   156|  </ul>
   157|{% endblock %}
   158|```
   159|
   160|常用标签：`{% if %}`, `{% for %}`, `{% url %}`, `{% csrf_token %}`  
   161|常用过滤器：`|safe`, `|date:"Y-m-d"`, `|upper`, `|default:"N/A"`
   162|
   163|---
   164|
   165|## 6. 表单处理（Forms）
   166|```python
   167|# forms.py
   168|from django import forms
   169|from .models import Article
   170|
   171|class ArticleForm(forms.ModelForm):
   172|    class Meta:
   173|        model = Article
   174|        fields = ["title", "content", "category", "tags", "status"]
   175|        widgets = {
   176|            "content": forms.Textarea(attrs={"rows": 6}),
   177|            "tags": forms.CheckboxSelectMultiple,
   178|        }
   179|
   180|    def clean_title(self):
   181|        title = self.cleaned_data.get("title")
   182|        if len(title) < 5:
   183|            raise forms.ValidationError("标题至少 5 个字符")
   184|        return title
   185|```
   186|
   187|```python
   188|# views.py
   189|from django.shortcuts import render, redirect
   190|from .forms import ArticleForm
   191|
   192|def create_article(request):
   193|    if request.method == "POST":
   194|        form = ArticleForm(request.POST)
   195|        if form.is_valid():
   196|            article = form.save(commit=False)
   197|            article.author = request.user
   198|            article.save()
   199|            form.save_m2m()  # 保存多对多关系
   200|            return redirect("article_list")
   201|    else:
   202|        form = ArticleForm()
   203|    return render(request, "core/article_form.html", {"form": form})
   204|```
   205|
   206|---
   207|
   208|## 7. URL 路由配置
   209|```python
   210|# config/urls.py
   211|from django.contrib import admin
   212|from django.urls import path, include
   213|
   214|urlpatterns = [
   215|    path("admin/", admin.site.urls),
   216|    path("api/v1/", include("api.urls")),
   217|    path("", include("core.urls")),
   218|]
   219|```
   220|
   221|```python
   222|# core/urls.py
   223|from django.urls import path
   224|from . import views
   225|
   226|app_name = "core"
   227|
   228|urlpatterns = [
   229|    path("articles/", views.ArticleListView.as_view(), name="article_list"),
   230|    path("articles/<int:pk>/", views.ArticleDetailView.as_view(), name="article_detail"),
   231|    path("articles/create/", views.create_article, name="article_create"),
   232|]
   233|```
   234|
   235|---
   236|
   237|## 8. 中间件（Middleware）原理
   238|中间件是请求/响应处理管道的钩子。Django 3.1+ 推荐使用 `__call__` 协议。
   239|
   240|```python
   241|# middleware.py
   242|import time
   243|from django.utils.deprecation import MiddlewareMixin
   244|
   245|class TimingMiddleware:
   246|    def __init__(self, get_response):
   247|        self.get_response = get_response
   248|
   249|    def __call__(self, request):
   250|        start = time.time()
   251|        response = self.get_response(request)
   252|        duration = time.time() - start
   253|        response["X-Process-Time"] = f"{duration:.4f}s"
   254|        return response
   255|
   256|# settings.py
   257|MIDDLEWARE = [
   258|    "django.middleware.security.SecurityMiddleware",
   259|    "django.contrib.sessions.middleware.SessionMiddleware",
   260|    "django.middleware.common.CommonMiddleware",
   261|    "django.middleware.csrf.CsrfViewMiddleware",
   262|    "django.contrib.auth.middleware.AuthenticationMiddleware",
   263|    "config.middleware.TimingMiddleware",  # 自定义中间件
   264|    "django.contrib.messages.middleware.MessageMiddleware",
   265|    "django.middleware.clickjacking.XFrameOptionsMiddleware",
   266|]
   267|```
   268|
   269|---
   270|
   271|## 9. Django REST Framework 集成
   272|```bash
   273|pip install djangorestframework django-filter
   274|```
   275|
   276|```python
   277|# settings.py
   278|INSTALLED_APPS = [..., "rest_framework", "django_filters"]
   279|REST_FRAMEWORK = {
   280|    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.PageNumberPagination",
   281|    "PAGE_SIZE": 20,
   282|    "DEFAULT_FILTER_BACKENDS": ["django_filters.rest_framework.DjangoFilterBackend"],
   283|}
   284|```
   285|
   286|```python
   287|# api/serializers.py
   288|from rest_framework import serializers
   289|from core.models import Article
   290|
   291|class ArticleSerializer(serializers.ModelSerializer):
   292|    author_name = serializers.CharField(source="author.username", read_only=True)
   293|
   294|    class Meta:
   295|        model = Article
   296|        fields = "__all__"
   297|        read_only_fields = ["author", "created_at"]
   298|```
   299|
   300|```python
   301|# api/views.py
   302|from rest_framework import viewsets
   303|from rest_framework.permissions import IsAuthenticatedOrReadOnly
   304|from core.models import Article
   305|from .serializers import ArticleSerializer
   306|
   307|class ArticleViewSet(viewsets.ModelViewSet):
   308|    queryset = Article.objects.filter(status="published")
   309|    serializer_class = ArticleSerializer
   310|    permission_classes = [IsAuthenticatedOrReadOnly]
   311|    filterset_fields = ["category", "status"]
   312|```
   313|
   314|```python
   315|# api/urls.py
   316|from rest_framework.routers import DefaultRouter
   317|from .views import ArticleViewSet
   318|
   319|router = DefaultRouter()
   320|router.register(r"articles", ArticleViewSet)
   321|
   322|urlpatterns = router.urls
   323|```
   324|
   325|---
   326|
   327|## 10. 数据库优化与查询
   328|避免 N+1 查询问题，合理使用 `select_related`（外键）和 `prefetch_related`（多对多/反向外键）。
   329|
   330|```python
   331|# ❌ 低效：每次循环触发一次 DB 查询
   332|articles = Article.objects.all()
   333|for a in articles:
   334|    print(a.category.name)  # N+1
   335|
   336|# ✅ 高效：JOIN 预加载
   337|articles = Article.objects.select_related("category", "author").all()
   338|for a in articles:
   339|    print(a.category.name)  # 1 次查询
   340|
   341|# ✅ 多对多预加载
   342|articles = Article.objects.prefetch_related("tags").all()
   343|for a in articles:
   344|    print([t.name for t in a.tags.all()])
   345|```
   346|
   347|其他优化手段：
   348|- 使用 `.values()`, `.values_list()` 返回轻量级结构
   349|- 为高频查询字段添加索引：`models.Index(fields=["status", "created_at"])`
   350|- 避免在视图中使用 `len(QS)`，改用 `QS.count()`
   351|
   352|---
   353|
   354|## 11. 安全最佳实践
   355|```python
   356|# settings.py 关键配置
   357|DEBUG = False
   358|ALLOWED_HOSTS = ["example.com", "*.example.com"]
   359|SECRET_KEY = os.environ.get("DJANGO_SECRET_KEY")
   360|
   361|# HTTPS 强制
   362|SECURE_SSL_REDIRECT = True
   363|SESSION_COOKIE_SECURE = True
   364|CSRF_COOKIE_SECURE = True
   365|SECURE_BROWSER_XSS_FILTER = True
   366|SECURE_CONTENT_TYPE_NOSNIFF = True
   367|X_FRAME_OPTIONS = "DENY"
   368|
   369|# 密码策略
   370|AUTH_PASSWORD_VALIDATORS = [
   371|    {"NAME": "django.contrib.auth.password_validation.MinimumLengthValidator"},
   372|    {"NAME": "django.contrib.auth.password_validation.CommonPasswordValidator"},
   373|]
   374|```
   375|- 始终使用 `render(request, template, context)` 自动转义输出
   376|- 表单提交必须包含 `{% csrf_token %}`
   377|- 敏感操作使用权限装饰器或 `@permission_required`
   378|- 定期运行 `python manage.py check --deploy`
   379|
   380|---
   381|
   382|## 12. 缓存与性能优化
   383|```python
   384|# settings.py
   385|CACHES = {
   386|    "default": {
   387|        "BACKEND": "django.core.cache.backends.redis.RedisCache",
   388|        "LOCATION": "redis://127.0.0.1:***@method_decorator(cache_page(60 * 15), name="dispatch")
   399|class ArticleListView(ListView):
   400|    ...
   401|
   402|# 模板片段缓存
   403|{% load cache %}
   404|{% cache 300 article_list user.id %}
   405|  <!-- 动态内容 -->
   406|{% endcache %}
   407|
   408|# 底层缓存 API
   409|from django.core.cache import cache
   410|cache.set("key", value, timeout=300)
   411|value = cache.get("key", default=None)
   412|cache.delete("key")
   413|```
   414|建议配合 `django-debug-toolbar` 分析慢查询与渲染耗时。
   415|
   416|---
   417|
   418|## 13. 测试策略
   419|Django 内置 `unittest` 支持，推荐使用 `pytest-django` 提升效率。
   420|
   421|```python
   422|# tests.py
   423|from django.test import TestCase, Client
   424|from django.contrib.auth.models import User
   425|from .models import Article, Category
   426|
   427|class ArticleModelTest(TestCase):
   428|    @classmethod
   429|    def setUpTestData(cls):
   430|        cls.cat = Category.objects.create(name="Tech")
   431|        cls.user = User.objects.create_user(username="testuser", password="pass123")
   432|
   433|    def test_create_article(self):
   434|        article = Article.objects.create(
   435|            title="Test", content="Body", author=self.user, category=self.cat
   436|        )
   437|        self.assertEqual(article.status, "draft")
   438|        self.assertTrue(article.created_at)
   439|
   440|class ArticleViewTest(TestCase):
   441|    def setUp(self):
   442|        self.client = Client()
   443|        self.user = User.objects.create_user("tester", "t@test.com", "pwd")
   444|        self.cat = Category.objects.create(name="Dev")
   445|        self.article = Article.objects.create(
   446|            title="Published", content="Content", author=self.user,
   447|            category=self.cat, status="published"
   448|        )
   449|
   450|    def test_article_list(self):
   451|        resp = self.client.get("/articles/")
   452|        self.assertEqual(resp.status_code, 200)
   453|        self.assertContains(resp, "Published")
   454|        self.assertQuerySetEqual(
   455|            resp.context["articles"], [self.article], transform=lambda x: x
   456|        )
   457|```
   458|- 使用 `TransactionTestCase` 需手动回滚或加 `@override_settings(USE_TZ=True)`
   459|- 测试数据库隔离：每次测试在独立事务中运行，不污染生产数据
   460|
   461|---
   462|
   463|## 14. 部署方案（Gunicorn + Nginx）
   464|### Gunicorn 配置 (`gunicorn.conf.py`)
   465|```python
   466|bind = "127.0.0.1:8000"
   467|workers = 4
   468|threads = 2
   469|timeout = 120
   470|keepalive = 5
   471|accesslog = "-"
   472|errorlog = "-"
   473|loglevel = "info"
   474|```
   475|
   476|### Systemd 服务 (`/etc/systemd/system/django.service`)
   477|```ini
   478|[Unit]
   479|Description=Gunicorn instance to serve Django
   480|After=network.target
   481|
   482|[Service]
   483|User=www-data
   484|Group=www-data
   485|WorkingDirectory=/opt/myproject
   486|ExecStart=/opt/venv/bin/gunicorn config.wsgi:application -c gunicorn.conf.py
   487|ExecReload=/bin/kill -s HUP $MAINPID
   488|Restart=on-failure
   489|
   490|[Install]
   491|WantedBy=multi-user.target
   492|```
   493|
   494|### Nginx 反向代理 (`/etc/nginx/sites-available/django`)
   495|```nginx
   496|server {
   497|    listen 80;
   498|    server_name example.com;
   499|    return 301 https://$host$request_uri;
   500|}
   501|
   502|server {
   503|    listen 443 ssl http2;
   504|    server_name example.com;
   505|
   506|    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
   507|    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
   508|
   509|    client_max_body_size 20M;
   510|
   511|    location /static/ {
   512|        alias /opt/myproject/static_collected/;
   513|        expires 30d;
   514|    }
   515|
   516|    location /media/ {
   517|        alias /opt/myproject/media/;
   518|        expires 30d;
   519|    }
   520|
   521|    location / {
   522|        proxy_pass http://127.0.0.1:8000;
   523|        proxy_set_header Host $host;
   524|        proxy_set_header X-Real-IP $remote_addr;
   525|        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
   526|        proxy_set_header X-Forwarded-Proto $scheme;
   527|    }
   528|}
   529|```
   530|
   531|部署流程：
   532|```bash
   533|python manage.py collectstatic --noinput
   534|python manage.py migrate
   535|systemctl restart django
   536|systemctl enable django
   537|```
   538|
   539|---
   540|本文档覆盖 Django 核心机制与现代工程实践，适用于中大型项目开发。实际使用时请结合业务场景调整配置，并持续关注官方文档更新。
   541|
   542|