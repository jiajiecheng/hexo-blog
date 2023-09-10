---
title: Django学习笔记(一)
date: 2023-09-10 17:19:01
tags: Django
---
## Django-admin后台

### 介绍

django提供了比较完善的后台管理数据库的接口，可供开发过程中调用和测试使用 django会搜集所有已注册的模型类为这些模型类提拱数据管理界面，供开发者使用

### 基本使用

#### 创建超级用户

通过命令创建一个超级用户，随后输入用户名，邮箱和密码回车即可，如下：

```pycon
python .\manage.py createsuperuser
```

创建成功：

```
Username (leave blank to use 'administrator'): root
Email address: root@qq.com
Password:
Password (again):
Superuser created successfully.
```

输入对应的url进入到`admin`管理界面：
[http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/)  
如下：  
![img.png](img_23_9_10/img.png)

#### 中文化界面

在settings文件中设置

```pycon
LANGUAGE_CODE = 'zh-hans'
TIME_ZONE = 'Asia/Shanghai'
```

效果:
![img_1.png](img_23_9_10/img_1.png)

#### 组与权限

我们可以将组比喻成为部门，通过赋予这些组一定的权限。然后让对应的用户入组，这样就可以不用单独给每一个用户权权限了。
例如我们可以像下面这么做：
![img_2.png](img_23_9_10/img_2.png)
我们添加了一个前端开发组，并且给这个组对应权限，如果下次某些用户是前端开发者，那么可以直接加入到这个组中。

#### 管理模型类

如果我们需要在admin后台中管理自己的模型类，那么需要做如下的操作

1. 在`admin.py`文件中导入对应的模型类
2. 导入注册的方法
3. 注册模型

代码如下：

```pycon
from django.contrib import admin
from .models import Author

# Register your models here.
admin.site.register(Author)
```

效果：  
![img_3.png](img_23_9_10/img_3.png)

#### 修改显示样式

如果我们没有做任何的操作，那么此时里面的数据会呈对象形式展示  
![img_4.png](img_23_9_10/img_4.png)  
如果想要优化显示我们只需要在model中添加`__str__`方法,如下：

```pycon
class Author(models.Model):
    name = models.CharField(max_length=10)
    password = models.CharField(max_length=16)
    telephone = models.CharField(max_length=16)

    def __str__(self):
        return f'{self.name}-{self.password}-{self.telephone}'
```

最终效果：
![img_5.png](img_23_9_10/img_5.png)

#### ModelAdmin 对象

作用:
为后台管理界面添加便于操作的新功能  
说明:
后台管理器类须继承自 django.contrib.admin 里的 ModelAdmin 类

1. 继承 ModelAdmin 类
2. 编写对应的操作代码
3. 注册的时候传递对应的模型已经模型管理类

```pycon
from django.contrib import admin
from .models import Author

class AuthorAdmin(admin.ModelAdmin):
    pass

admin.site.register(Author, AuthorAdmin)
```

#### 使用register装饰器

对于上面的操作我们可以使用更加简便的装饰器来完成  
代码如下：

```pycon
from django.contrib import admin
from .models import Author

@admin.register(Author)
class AuthorAdmin(admin.ModelAdmin):
    pass
```

#### ModelAdmin类属性

##### list_display

添加字段分类

```pycon
@admin.register(Author)
class AuthorAdmin(admin.ModelAdmin):
    list_display = ['id', 'name', 'password', 'telephone']
```

效果：
![img_6.png](img_23_9_10/img_6.png)
需要注意的是此时添加的名字必须和模型字段名相同，如果想要显示中文信息，请在模型类字段中添加`verbose_name`参数

```pycon
class Author(models.Model):
    name = models.CharField(verbose_name='书名', max_length=10)
    password = models.CharField(verbose_name='密码', max_length=16)
    telephone = models.CharField(verbose_name='电话', max_length=16)
```

效果：
![img_7.png](img_23_9_10/img_7.png)

##### list_display_links

用于设置点哪个字段会进入到修改页

```pycon
....
    list_display_links = ['id']
....
```

效果
![img_8.png](img_23_9_10/img_8.png)

##### list_filter

设置可以用来的分类的字段

```pycon
....
    list_filter = ['name']
....
```

效果
![img_9.png](img_23_9_10/img_9.png)

##### search_fields

设置允许搜索的字段

```pycon
....
    search_fields = ['id']
....
```

效果
![img_10.png](img_23_9_10/img_10.png)

##### list_editable

添加允许直接在列表中直接修改的字段

```pycon
....
    list_editable = ['telephone']
....
```

效果：
![img_11.png](img_23_9_10/img_11.png)

##### 更多

更多字段的作用请查看的官网文档  
[https://docs.djangoproject.com/zh-hans/4.2/ref/contrib/admin/#modeladmin-objects](https://docs.djangoproject.com/zh-hans/4.2/ref/contrib/admin/#modeladmin-objects)

#### Meta与ModelAdmin类的联动

##### db_table

设置表名

##### verbose_name

设置表名称，能够在admin后台展示

```pycon
.....
    class Meta:
        db_table = 'author'
        verbose_name = '作者'
```

效果
![img_12.png](img_23_9_10/img_12.png)

##### verbose_name_plural

上面由于使用了复数形式，所以名称后面带上了s,故为`作者s`，如果我们需要设置这个表复数状态下的名称我们可以使用：

```pycon
.....
    class Meta:
        db_table = 'author'
        verbose_name = '作者'
        verbose_name_plural = verbose_name
```

效果：
![img_13.png](img_23_9_10/img_13.png)
