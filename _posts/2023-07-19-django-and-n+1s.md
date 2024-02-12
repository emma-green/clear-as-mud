---
layout: post
date:   2023-07-19 12:30:11 -0400
categories: django
author: Emma
---

Sometimes finding an N+1 in your code is like finding gold -- you have an easy performance win. Other times...it'll ruin your day. Hopefully, this article will help you to destroy those pesky N+1s!

## So, what's an N+1?

An N+1 is anytime that you make a query in a loop. Literally doing it N times. The +1 comes from the first query that you did to find the elements that you are looping over.

**Clear As Mud??**


Let's say you are making an app where you want to list all of the students.

**Student Table**

| Column Name   | Notes                     |
|---------------|---------------------------|
| id            | primary key               |
| first_name    |                           |
| last_name     |                           |


You have a function that takes in a list of student ids, and prints all of the information needed for each of them.

You could loop through each id and get the info:
{% highlight python %}
for student_id in student_ids:
    cursor.execute("SELECT * FROM STUDENT where student_id = %s", [student_id])
    print(cursor.fetchall())
{% endhighlight %}

As the number (N) of students grow, so too does the number of queries you are executing.
Ten students might not seem like a lot, but we want our app to be able to handle thousands,
and eventually millions of students. And ten queries might actually be a lot depending on our infrastructure.

Instead of looping through each id, we can make one query for all of the students:
{% highlight python %}
cursor.execute("SELECT * FROM STUDENT where student_id IN %s", [student_ids])
print(cursor.fetchall())
{% endhighlight %}

### And that's it! That's all there is to N+1's. 

Yeah, I know. It can't really be that easy. And it's not. They like to hide from you in the code and then explode when you try to scale.

## Let's take a look at N+1s in Django

If we were to take that simple example from before, it would look like this using Django's ORM:
{% highlight python %}
for student_id in student_ids:
    student = Student.objects.get(id=student_id)
    print(student)
{% endhighlight %}

And to remove the N+1, we'd do something like this:
{% highlight python %}
students = Student.objects.filter(id__in=student_ids)
for student in students:
    print(student)
{% endhighlight %}

You'll notice that both code snippets include a loop. All we did was move the query out of the loop.

```
💡 TIP 💡
Run a database query and then loop over the results
instead of running a query in a loop.
```

### Most of the time, N+1s are not going to be quite so obvious.

This code has an N+1. Can you spot it?

models.py
{% highlight python %}
from django.db import models

class Reporter(models.Model):
    full_name = models.CharField(max_length=70)

    def __str__(self):
        return self.full_name


class Article(models.Model):
    pub_date = models.DateField()
    headline = models.CharField(max_length=200)
    content = models.TextField()
    reporter = models.ForeignKey(Reporter, on_delete=models.CASCADE)

    def __str__(self):
        headline = self.headline
        reporter_name = self.reporter.full_name
        return f'{headline} by {reporter_name}'
{% endhighlight %}

admin.py
{% highlight python %}
from django.contrib import admin
from .models import Article, Reporter

@admin.register(Article)
class ArticleAdmin(admin.ModelAdmin):
    pass
{% endhighlight %}

At first glace, this code doesn't look like it has a loop, right? If you search for `for` or `while`, you won't find it. However, the Django Admin list page for Article will execute a query for every article.

The query is in this line:
{% highlight python %}
reporter_name = self.reporter.full_name
{% endhighlight %}

Spot it now?

That line runs a query to get the reporter associated with the current article.

The loop is because Django Admin uses the `__str__` method for displaying objects if we don't pass any information about what fields to display. It will call the `__str__` method for every article.

```
💡 TIP 💡
Do not get information from related models
in the __str__ method.
```
## Other Common Django N+1 Pitfalls

### Accessing a related object/model in a property
models.py
{% highlight python %}
from django.db import models

class Reporter(models.Model):
    full_name = models.CharField(max_length=70)

    def __str__(self):
        return self.full_name

    @property
    def most_recent_article(self):
        return self.article_set.latest('pub_date')


class Article(models.Model):
    pub_date = models.DateField()
    headline = models.CharField(max_length=200)
    content = models.TextField()
    reporter = models.ForeignKey(Reporter, on_delete=models.CASCADE)

    def __str__(self):
        return self.headline

{% endhighlight %}

Properties are referenced like attributes, which means that we would use `reporter.most_recent_article` instead of `reporter.most_recent_article()`. Therefore, it's hard to know just from the method signature that it actually accesses the db to find the most recent article and could lead to incorrect usage.

```
💡 TIP 💡
Do not use a property decorator for a method that will 
access information from related models.
```
### Returning unnecessary information in a method
models.py
{% highlight python %}
from django.db import models

class Reporter(models.Model):
    full_name = models.CharField(max_length=70)

    def __str__(self):
        return self.full_name

    def get_most_recent_article(self):
        return self.article_set.latest('pub_date')

    def get_all_reporter_info(self):
        return {
            'full_name': self.full_name, 
            'most_recent_article': self.get_most_recent_article(),
        }


class Article(models.Model):
    pub_date = models.DateField()
    headline = models.CharField(max_length=200)
    content = models.TextField()
    reporter = models.ForeignKey(Reporter, on_delete=models.CASCADE)

    def __str__(self):
        return self.headline

{% endhighlight %}

You may be tempted to write and use a method like `get_all_reporter_info` because then you know that the info will be returned the same every time. However, if you could end up in a situation where you don't actually need to get information from a related table, but do so anyway.

```
💡 TIP 💡
Do not write "one-size-fits-all" methods.
```

## Avoiding N+1s in Django when Accessing Another Model is Necessary
So far, we've discused how to avoid accessing related objects. Sometimes you will legitimately need to access related objects. Django gives us a couple of major tools for doing this.


### Using select_related
Django has a function called `select_related`. This function will put a related record in the cache. 

Let's look at this in action

models.py
{% highlight python %}
from django.db import models

class Employee(models.Model):
    full_name = models.CharField(max_length=70)


class Reporter(models.Model):
    biline_name = models.CharField(max_length=70)
    reporter = models.ForeignKey(Reporter, on_delete=models.CASCADE)

    def __str__(self):
        return self.biline_name

    def get_most_recent_article(self):
        return self.article_set.latest('pub_date')


class Article(models.Model):
    pub_date = models.DateField()
    headline = models.CharField(max_length=200)
    content = models.TextField()
    reporter = models.ForeignKey(Reporter, on_delete=models.CASCADE)

    def __str__(self):
        return self.headline

{% endhighlight %}

Let's say we were in an area of code where we have an article id and we want to print the reporter's name.

{% highlight python %}
article = Article.objects.get(article_id)
reporter = article.reporter
employee = reporter.employee
name = employee.full_name
print(name)
{% endhighlight %}

(This could also be written as `print(article.get(article_id).reporter.employee.full_name)`. It will make the name number of queries. I've written it out in a longer fashion to be able to talk about each component part.)

The first three lines each make a new database request. 
1. First it selects the row in the article table that corresponds to the id. Let's pretend the value of article_id is 1
` SELECT * FROM Article WHERE id = 1` 
We get a result of:
```
--------------------------------------------------------
id | pub_date | headline   | content      | reporter_id
--------------------------------------------------------
1   2024-01-01  Chiefs Win   The chiefs..   2
```
2. then it selects the row in the reporter table that correlates to the foriegn key 
` SELECT * FROM Reporter WHERE id = 2` 
```
--------------------------------------------------------
id | biline_name | employee_id
--------------------------------------------------------
2    Sam Adams     3
```
2. finally, it selects the row in the employee table that correlates to the foriegn key 
` SELECT * FROM Employee WHERE id = e` 
```
--------------------------------------------------------
id | full_name 
--------------------------------------------------------
3    Samuel Adams  
```
Instead of three database requests, we could just do one with `select_related`
{% highlight python %}
article = Article.objects.select_related("reporter__employee").get(article_id)
reporter = article.reporter
employee = reporter.employee
name = employee.full_name
print(name)
{% endhighlight %}

Now, there's only one database query, which is on the first line. It looks like this:
` SELECT * FROM Article JOIN Reporter ON Article.reporter_id = Reporter.id JOIN Employee ON Reporter.employee_id = Employee.id WHERE Article.id = 1` 
```
----------------------------------------------------------------------------------------------------------------
id | pub_date | headline   | content      | reporter_id | id | biline_name | employee_id | id | full_name
----------------------------------------------------------------------------------------------------------------
1   2024-01-01  Chiefs Win   The chiefs..   2              2   Sam Adams     3             3    Samuel Adams  

This query is much more complex than each of the original queries. It is *usually* a performance enhancement to have one complex query rather than a few less complex queries, but not always. It will depend on your system and infrastructure.

Another interesting thing to note here is that because the related objects are put in the cache, anywhere in our code that we access them will provide the same result and will not go to the database. This could mean stale data.

A deep bug to look out for here: 
{% highlight python %}
def print_employee_name(article)
    reporter = article.reporter
    employee = reporter.employee
    name = employee.full_name
    print(name)

def print_employee_name_from_article_id_A(article_id)
    article = Article.objects.select_related("reporter__employee").get(article_id)
    print_employee_name(article)

def print_employee_name_from_article_id_B(article_id)
    article = Article.objects.get(article_id)
    print_employee_name(article)
    
{% endhighlight %}

By looking at the `print_employee_name` function, we might assume that there are two database calls made. However, if our code stack includes the print_employee_name_from_article_id_A function, the data might already be cached. If we have a code path that sometimes calls the A function and sometimes calls the B function, the data might sometimes be cached.

How is `select_related` useful for N+1's?
So far, we've only looked at examples where we are getting one record and it's related records. N+1's are found in loops. So, let's use `select_related` for a loop.

{% highlight python %}
articles = Article.objects.get(id__in=article_ids)
for article in articles:
    reporter = article.reporter
    employee = reporter.employee
    name = employee.full_name
    print(name)
{% endhighlight %}

`article.reporter` and `reporter.employee` create N+1s, because we are doing two extra queries for each article.

Instead, we can do this:
{% highlight python %}
articles = Article.objects.get(id__in=article_ids).select_related("reporter__employee")
for article in articles:
    reporter = article.reporter
    employee = reporter.employee
    name = employee.full_name
    print(name)
{% endhighlight %}
Now, we only do one query.


