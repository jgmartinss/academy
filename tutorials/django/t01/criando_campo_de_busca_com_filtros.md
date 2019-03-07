# Criando um Campo de Busca com Filtros
> Nesse tutorial irei mostrar como criar um campo de busca com filtros em Django, de forma bem simples. Colocando em prática os seguintes conceitos;

  - [Class Based Views](https://docs.djangoproject.com/en/2.1/topics/class-based-views/)
  - [Managers](https://docs.djangoproject.com/pt-br/2.1/topics/db/managers/)
  - [Mixins](https://docs.djangoproject.com/en/2.1/topics/class-based-views/mixins/)

## Criando o projeto 

Antes de tudo precisamos criar um ambiente virtual para criarmos o projeto, vou utilizar o [pipenv](https://github.com/pypa/pipenv) (Caso não o conheça, leia esse post do [@hudsonbrendon](https://medium.com/grupy-rn/gerenciando-suas-depend%C3%AAncias-e-ambientes-python-com-pipenv-9e5413513fa6)) e instale as dependências usando os seguintes comandos:

```sh
$ pipenv install django==2.1.7
```
Crie o projeto executando o comando abaixo:

```sh
$ django-admin.py startproject myproject .
```

Para demosntrar a funcionalidade irei criar uma app chamada 'posts'

```sh
$ django-admin.py startapp posts
```

## Configurando o settings

Configure o seu settings como o exemplo abaixo:

```py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
 
    'posts',
]

TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [os.path.join(BASE_DIR, 'templates')],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]
```

## Criando o models

Com o arquivo settings configurado, agora podemos criar nossos models no arquivo posts/models.py Vamos criar as seguintes entidades, Category e Post. Como o código abaixo:

```py
from django.db import models


class Category(models.Model):
    name = models.CharField("Name", max_length=50)

    class Meta:
        verbose_name = "Category"
        verbose_name_plural = "Categorys"

    def __str__(self):
        return self.name


class Post(models.Model):
    title = models.CharField("Title", max_length=50)
    body = models.TextField("Body")
    category = models.ForeignKey(
        Category, verbose_name="Category", on_delete=models.CASCADE
    )
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)


    class Meta:
        verbose_name = "Post"
        verbose_name_plural = "Posts"
        ordering = ["-created_at"]

    def __str__(self):
        return self.title

```

Com os models definidos precisamos migrar as novas entidades e criar o banco de dados.

```sh
$ python manage.py makemigrations

$ python manage.py migrate
```

## Configurando o Admin

Agora precisamos configurar nosso admin, no arquivo posts/admin.py
Como o código abaixo:

```py
from django.contrib import admin

from . import models


class PostAdmin(admin.ModelAdmin):
    model = models.Post
    list_display = ('title', 'category',)
    list_filter = ('category',)


admin.site.register(models.Category)
admin.site.register(models.Post, PostAdmin)

```

Com isso o Django cria um <strong><i>CRUD</i></strong> bem simples do nossos models e disponibiliza o mesmo em seu admin. Para exemplificar vamos criar um super usuário e visualizar o resultado, para isso execute:

```sh
$ python manage.py createsuperuser
```

Preencha os dados solicitados, e execute:

```sh
$ python manage.py runserver
```
Acesse seu localhost:8000/admin/, realize o login e você terá uma tela como essa:

![img1](https://raw.github.com/jgmartinss/academy/master/tutorials/django/t01/img/img1.png)

Agora que temos o banco criado, vamos inserir alguns registros.

Nas seguintes urls, você terá uma tela semelhante a essa:
- localhost:8000/admin/posts/category/

![img2](https://raw.github.com/jgmartinss/academy/master/tutorials/django/t01/img/img2.png)

- localhost:8000/admin/posts/post/

![img3](https://raw.github.com/jgmartinss/academy/master/tutorials/django/t01/img/img3.png)


## Criando o Managers

Para facilitar nossas consultas iremos criar nossas classes QuerySets no arquivo posts/manager.py. Como o código abaixo:

```py
from django.db import models


class PostQuerySet(models.QuerySet):
    def search(self, term_one, term_two):
        if not term_one and term_two:
            return self

        params = models.Q(title__icontains=term_one) and models.Q(category__id=term_two)
        return self.filter(params)

```

Assim podemos filtrar usando múltiplos parametros em nossa query, ficando mais compacta e coesa. Com o manager feito temos que adicionalo ao nosso model Post, em posts/model.py logo abaixo dos fields de nossa entidade. Como o código abaixo:

```py
from django.db import models

from . import managers


class Post(models.Model):
    ...
    # fields

    objects = models.Manager.from_queryset(managers.PostQuerySet)()

    class Meta:
        ...

    def __str__(self):
        return self.title

```

Agora nossa entidade Post tem acesso ao filtro que criamos acima, podemos utilizar dessa maneira.

```py
Post.objects.search('', '')

```

## Criando o Mixins

Vamos criar o nosso PostMixin em posts/mixins.py. Neles iremos abstrair as consultas das views, são responsáveis por tratarem a manipulação de dados que a view terá que realizar. Neste caso nosso mixin será responsável por coletar os dados enviados via request, através do método GET. Como o código abaixo:

```py
from . import models


class PostMixin(object):
    def get_queryset(self):
        super(PostMixin, self).get_queryset()
        
        term_one = self.request.GET.get("search_box")
        term_two = self.request.GET.get("search_param")

        return models.Post.objects.search(term_one, term_two)

```

## Criando as views

Com os models, managers e mixins prontos, agora podemos criar as views. Primeiramente iremos criar uma view que retorna uma lista de categorias, e outra para retornar uma lista de posts conforme o que foi pesquisado.

```py

from django.views import generic

from . import models
from . import mixins


class IndexView(generic.TemplateView):
    template_name = "index.html"

    def get_context_data(self, **kwargs):
        context = super(IndexView, self).get_context_data(**kwargs)
        context["categorys"] = models.Category.objects.all().order_by("name")
        return context


class PostListView(mixins.PostMixin, generic.ListView):
    model = models.Post
    context_object_name = "posts"
    template_name = "post_list.html"

```

Na view PostListView estamos utilizando nosso mixin.

## Criando as urls

Temos que criar nossas urls em posts/urls.py 
Como o código a baixo:

```py
from django.urls import path

from . import views


app_name = "posts"

urlpatterns = [
    path("", views.IndexView.as_view(), name="index"),
    path("posts/", views.PostListView.as_view(), name="search-posts"),
]
```

Agora em myproject/urls.py precisamos chamar as urls criadas, para que o django consiga identificá-las:

```py
from django.contrib import admin
from django.urls import path, include


urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('posts.urls')),
]

```

Com as views prontas, e urls configuradas temos que criar os seus respectivos templates. Primeiramente vamos criar um template genérico para pouparmos tempo com a criação de novos arquivos html, chamado de base.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Criando campo de busca com filtros</title>
    <script src="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.5/js/bootstrap.min.js"></script>
    <link href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.5/css/bootstrap-theme.min.css" rel="stylesheet" />
    <link href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.5/css/bootstrap.min.css" rel="stylesheet" />
</head>
<body>
    {% block content %}
    {% endblock content %}
</body>
</html>
```
O segundo template é nossa index.html, onde vamos carregar nosso campo de busca com seus respectivos filtros.

```html
{% extends 'base.html' %}
{% block content %}
    <div class="col-md-9 col-md-push-1"><br>
        <div class="container">
            <div class="row">
                <div class="col-xs-8 col-xs-offset-2">
                    <form action="{% url 'posts:search-posts' %}" method="get" id="searchForm" class="input-group">
                        <div class="input-group-btn search-panel">
                            <select name="search_param" id="search_param" class="btn btn-default dropdown-toggle"
                                data-toggle="dropdown">
                                {% if categorys %}
                                    {% for category in categorys %}
                                        <option value="{{category.id}}">{{category.name}}</option>
                                    {% endfor %}
                                {% else %}
                                    <option value="0">--------</option>
                                {% endif %}
                            </select>
                        </div>
                        <input type="text" class="form-control" name="search_box" placeholder="Buscar...">
                        <span class="input-group-btn">
                            <button class="btn btn-default" type="submit">
                                <span class="glyphicon glyphicon-search"></span>
                            </button>
                        </span>
                    </form>
                </div>
            </div>
        </div>
    </div><br><br>
{% endblock %}
```

O terceiro template é post_list.html, nele será renderizado uma lista de posts que nossa view retornará.

```html
{% extends 'base.html' %}
{% block content %}

    <h2>Resultados</h2>

    {% if posts %}
        {% for post in posts %}
            <h2>{{post.title}}</h2>
            <p>{{post.body}}</p>
            <p>{{post.category}}</p>
        {% endfor %}
    {% else %}
        <p>Nenhum post encontrado!</p>
    {% endif %}

{% endblock %}
```

Acesse seu localhost:8000/, e sua view IndexView irá retornar uma tela como essa:

![img4](https://raw.github.com/jgmartinss/academy/master/tutorials/django/t01/img/img4.png)


Se fizermos essa seguinte pesquisa.

![img5](https://raw.github.com/jgmartinss/academy/master/tutorials/django/t01/img/img5.png)

Vamos obter o seguinte resultado.

![img6](https://raw.github.com/jgmartinss/academy/master/tutorials/django/t01/img/img6.png)

E nossa url ficara assim após o resultado da pesquisa realizada.

![img7](https://raw.github.com/jgmartinss/academy/master/tutorials/django/t01/img/img7.png)


### Nem foi tão complicado neh! Gostou? Vamos espalhá-lo!!