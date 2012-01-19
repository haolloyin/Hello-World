#### 模型
----

###### 直连数据库

    from django.shortcuts import render_to_response
    import MySQLdb

    def book_list(request):
        db = MySQLdb.connect(user='me', db='mydb', passwd='secret', host='localhost')
        cursor = db.cursor()
        cursor.execute('SELECT name FROM books ORDER BY name')
        names = [row[0] for row in cursor.fetchall()]
        db.close()
        return render_to_response('book_list.html', {'names': names})

###### Django的Model方式

    from django.shortcuts import render_to_response
    from mysite.books.models import Book

    def book_list(request):
        books = Book.objects.order_by('name')
        return render_to_response('book_list.html', {'books': books})

###### MVC & MTC

Django 紧紧地遵循这种MVC模式(Model-View-Controller)，可以称得上是一种MVC框架。以下是Django中M、V和C各自的含义：

- M，数据存取，由django数据库层处理
- V，选择显示哪些数据要显示以及怎样显示，由视图和模板处理
- C，根据用户输入指派视图，由Django框架根据URLconf设置，对给定URL调用适当的Python函数

由于Controller由框架自行处理，而Django里更关注的是模型(Model)、模板(Template)和视图(Views)，Django也被称为MTV框架：

- M 代表模型(Model)，数据存取层。该层处理与数据相关的所有事务：如何存取、如何验证有效性、包含哪些行为以及数据之间的关系等
- T 代表模板(Template)，表现层。该层处理与表现相关的决定：如何在页面或其他类型文档中进行显示
- V 代表视图(View)，业务逻辑层。该层包含存取模型及调取恰当模板的相关逻辑，可看作模型与模板之间的桥梁

###### 数据库配置

在`settings.py`文件下配置对应的项：

    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.', # Add 'postgresql_psycopg2', 'postgresql', 'mysql', 'sqlite3' or 'oracle'.
            'NAME': '',                      # Or path to database file if using sqlite3.
            'USER': '',                      # Not used with sqlite3.
            'PASSWORD': '',                  # Not used with sqlite3.
            'HOST': '',                      # Set to empty string for localhost. Not used with sqlite3.
            'PORT': '',                      # Set to empty string for default. Not used with sqlite3.
        }
    }

###### project & app

> 一个project包含很多个Django app以及对它们的配置。
技术上，project的作用是提供配置文件，比方说哪里定义数据库连接信息, 安装的app列表，TEMPLATE_DIRS，等等。
一个app是一套Django功能的集合，通常包括模型和视图，按Python的包结构的方式存在。
Django本身内建有一些app，例如注释系统和自动管理界面。app的一个关键点是它们是很容易移植到其他project和被多个project复用。

**如果使用了Django的数据库层（模型）必须用`python manage.py startapp app_name`创建一个Django app**

Django模型是用Python代码形式表述的数据在数据库中的定义，Django用模型在后台执行SQL代码并把结果用Python的数据结构来描述，也使用模型来呈现SQL无法处理的高级概念。

###### Django Model

对于_多对多_关系，Django创建了一个额外的表（多对多连接表）来处理这种映射关系。

    from django.db import models

    class Publisher(models.Model):
        name = models.CharField(max_length=30)
        address = models.CharField(max_length=50)
        city = models.CharField(max_length=60)
        state_province = models.CharField(max_length=30)
        country = models.CharField(max_length=50)
        website = models.URLField()

    class Author(models.Model):
        first_name = models.CharField(max_length=30)
        last_name = models.CharField(max_length=40)
        email = models.EmailField()

    class Book(models.Model):
        title = models.CharField(max_length=100)
        authors = models.ManyToManyField(Author)
        publisher = models.ForeignKey(Publisher)
        publication_date = models.DateField()

**安装&激活 Model**

创建好`models.py`之后，在`settings.py`的`INSTALLED_APPS`中添加对应的`'myproject.myapp'`，并用`python manage.py validate`检查`models.py`中是否有语法或逻辑错误，若不是输出`0 errors found`则需要检查`models.py`以及`INSTALLED_APPS`是否有错误。

**生成sql**

执行`python manage.py sqlall myapp`将输出`models.py`对应的建表语句：

    BEGIN;
    CREATE TABLE "books_publisher" (
        "id" serial NOT NULL PRIMARY KEY,
        "name" varchar(30) NOT NULL,
        "address" varchar(50) NOT NULL,
        "city" varchar(60) NOT NULL,
        "state_province" varchar(30) NOT NULL,
        "country" varchar(50) NOT NULL,
        "website" varchar(200) NOT NULL
    )
    ;
    CREATE TABLE "books_author" (
        "id" serial NOT NULL PRIMARY KEY,
        "first_name" varchar(30) NOT NULL,
        "last_name" varchar(40) NOT NULL,
        "email" varchar(75) NOT NULL
    )
    ;
    CREATE TABLE "books_book" (
        "id" serial NOT NULL PRIMARY KEY,
        "title" varchar(100) NOT NULL,
        "publisher_id" integer NOT NULL REFERENCES "books_publisher" ("id") DEFERRABLE INITIALLY DEFERRED,
        "publication_date" date NOT NULL
    )
    ;
    CREATE TABLE "books_book_authors" (
        "id" serial NOT NULL PRIMARY KEY,
        "book_id" integer NOT NULL REFERENCES "books_book" ("id") DEFERRABLE INITIALLY DEFERRED,
        "author_id" integer NOT NULL REFERENCES "books_author" ("id") DEFERRABLE INITIALLY DEFERRED,
        UNIQUE ("book_id", "author_id")
    )
    ;
    CREATE INDEX "books_book_publisher_id" ON "books_book" ("publisher_id");
    COMMIT;

**同步Model到数据库**

执行`python manage.py syncdb`，Django会将Model对应的建表语句同步到数据库（通过`INSTALLED_APPS`配置查找数据库），但是*syncdb并不能将模型的修改或删除同步到数据库*；如果你修改或删除了一个模型，并想把它提交到数据库，syncdb并不会做出任何处理。重复执行syscdb并不会造成影响，因为没有添加新的model或app。

syncdb的输出结果：

    Creating table books_publisher
    Creating table books_author
    Creating table books_book_authors
    Creating table books_book
    Installing index for books.Book_authors model
    Installing index for books.Book model
    No fixtures found.

执行`python manage.py dbshell`可查看数据库的表结构、数据，以及备份等。

**数据添加**

- 只有执行了Model子类的save()方法才会将创建的model对象保存到数据库中，或者直接用`model.objects.create()`方法在创建model时将对象保存到数据库
- Django模型中每次修改字段值（即model对象对应表的某一列）之后，都必须调用`save()`方法进行保存，且所有保存都是针对*所有字段*进行update操作的  

		from books.models import Publisher
    	p1 = Publisher(name='Apress', address='2855 Telegraph Avenue',
        	 city='Berkeley', state_province='CA', country='U.S.A.',
         	 website='http://www.apress.com/')
    	p1.save()
    	p2 = Publisher(name="O'Reilly", address='10 Fawcett St.',
         	city='Cambridge', state_province='MA', country='U.S.A.',
         	website='http://www.oreilly.com/')
	    p2.save()
    	publisher_list = Publisher.objects.all()
	    publisher_list
	    [<Publisher: Publisher object>, <Publisher: Publisher object>]

**查看数据库信息**

利用`python manage.py dbshell`的`.dump table_name`可查看对应表的结构和数据：

    sqlite> .dump books_publisher
    BEGIN TRANSACTION;
    CREATE TABLE "books_publisher" (
        "id" integer NOT NULL PRIMARY KEY,
        "name" varchar(30) NOT NULL,
        "address" varchar(50) NOT NULL,
        "city" varchar(60) NOT NULL,
        "state_province" varchar(30) NOT NULL,
        "country" varchar(50) NOT NULL,
        "website" varchar(200) NOT NULL
    );
    INSERT INTO "books_publisher" VALUES(1,'Apress','2855 Telegraph Avenue','Berkeley','CA','U.S.A.','http://www.apress.com/');
    INSERT INTO "books_publisher" VALUES(2,'O''Reilly','10 Fawcett St.','Cambridge','MA','U.S.A.','http://www.oreilly.com/');
    COMMIT;

利用`python manage.py dbshell`的`.schema table_name`可查看对应表的结构：

    sqlite> .schema books_book_authors
    CREATE TABLE "books_book_authors" (
        "id" integer NOT NULL PRIMARY KEY,
        "book_id" integer NOT NULL,
        "author_id" integer NOT NULL REFERENCES "books_author" ("id"),
        UNIQUE ("book_id", "author_id")
    );
    CREATE INDEX "books_book_authors_752eb95b" ON "books_book_authors" ("book_id");
    CREATE INDEX "books_book_authors_cc846901" ON "books_book_authors" ("author_id");

**解析model对象**

为每一个model子类添加`__unicode__`方法，使之以Unicode方式显示出来。

    def __unicode__(self):
        return self.name
    或
    def __unicode__(self):
        return u'%s %s' % (self.first_name, self.last_name)

    >>> from books.models import Publisher
    >>> print Publisher.objects.all()
    [<Publisher: Apress>, <Publisher: O'Reilly>]

    否则显示如下：
    >>> publisher_list = Publisher.objects.all()
    >>> publisher_list
    [<Publisher: Publisher object>, <Publisher: Publisher object>]

**数据查找**

- `Model.objects`称为*管理器*，它能够管理数据查询的表格级操作
- `Model.objects.all()`的返回是一个`QuerySet`对象保存到数据库中
- `Model.objects.filter(field_1='', field_2=)`可过滤数据，返回记录集（列表），相当于`select ... where .. and ..`语句，`field__contains='xx'`则类似于 `LIKE %xx%`，还有icontains，startswith，endswith，range等
- `Model.objects.get(field='')`返回单个数据对象，如果结果为多个或零个则抛`MultipleObjectsReturned`和`DoesNotExist`异常，其中`DoesNotExist`是model对象的属性

**数据排序**

- `Model.object.order_by(field_1, field_2)`升序
- `Model.objects.order_by(-field_1)`在字段名前面加负号-表示降序
- 可在Model子类中定义`class Meta`，指定其ordering属性默认为['filed']则可提供默认排序方式

**数据返回**

- 查找和排序的结果可用列表切片的方式来返回，但不支持负索引（会抛出AssertionError异常）
- 用order_by('-field')[0]即可实现类似列表负索引的切片

**数据更新**

- 用QuerySet.update('field')方法更新指定字段的值，而不是更新所有列。

    	Publisher.objects.filter(id=52).update(name='Apress Publishing')
    	Publisher.objects.all().update(country='USA')

**数据删除**

- 调用对象的`delete()`方法即可删除单个数据
- 显式调用Model.objects.all()或filter()的delete()方法将删除对应QuerySet保存的所有数据
- 不能直接对Model.objects调用delete()，抛`AttributeError: 'Manager' object has no attribute 'delete'`



