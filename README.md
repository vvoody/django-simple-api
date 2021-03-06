# Django Simple API

A non-intrusive component that can help you quickly create APIs.

## Install

Download and install from github

```
pip install git+https://github.com/abersheeran/django-simple-api.git@setup.py
```

Or from coding mirror in China

```
pip install git+https://e.coding.net/aber/github/django-simple-api.git@setup.py
```

Add django-simple-api to your `INSTALLED_APPS` in settings:

```python
INSTALLED_APPS = [
    ...
    "django_simple_api",
]
```

Add `SimpleApiMiddleware` to your `MIDDLEWARE` in settings:

```python
MIDDLEWARE = [
    ...
    "django_simple_api.middleware.SimpleApiMiddleware",
]
```

## Usage

⚠️ We support both `view-function` and `class-view` at the same time for all functions. If there is no special description in the document, it means that it is applicable to both views. Where special support is needed, we will indicate how to do it in the document.

### Parameter declaration and verification

Simple API use `pydantic` to declare parameters and parameter verification.

You can declare request parameters like the following example:

```python
# views.py
from django.views import View
from django.http.response import HttpResponse

from django_simple_api import Query

class JustTest(View):
    def get(self, request, id: int = Query(...)):
        return HttpResponse(id)
```

Simple API has a total of 6 fields, corresponding to the parameters in different positions:

##### All fields and description
| Field     | Description |
| ---       | ---         |
| Query     | Indicates that this parameter is in the url query string. example: http://host/?param=1|
| Path      | Indicates that this parameter is a url path parameter. example: http://host/{param}/|
| Body      | Indicates that this parameter is in the request body, and the value can only be obtained in a non-GET request.|
| Cookie    | Indicates that this parameter is in Cookie.|
| Header    | Indicates that this parameter is in Header.|
| Exclusive | This is a special field, its parameter annotation should be subclass of `pydantic.BaseModel`, it will get all the parameters from the location specified by `Exclusive`.

For example:

```python
from pydantic import BaseModel, Field

class ArticleForm(BaseModel):
    article_title: str = Field(...)
    article_content: str = Field(...)


class JustTest(View):
    # The parameter names used in the above examples are for demonstration only.
    def post(self, request,
            param1: int = Query(...),
            param2: int = Query(...),
            param3: int = Path(...),
            # param4: str = Body(...),
            userid: int = Cookie(..., alias="uid"),
            csrf_token: str = Header(..., alias="X-CSRF-TOKEN"),

            # Simple API will get the `article_title` `article_content` parameter from the request body and create an object `article`
            article: ArticleForm = Exclusive("body"),
        ):

        # You can get the parameters like this:
        title = article.article_title
        content = article.article_content

        # Or directly convert to a dictionary:
        d = article.dict()  # {"article_title": ..., "article_content": ...}
        return HttpResponse(d)
```

⚠️ In the above example, you have two things to note:
* When you need to get parameters from `Header`, you may need to use `alias` to indicate the request header you want to get, because the name of the request header may not be a valid python identifier.
* When you use `Exclusive("body")` to get the form from a specified location, you can no longer use the `Body` field.

As you can see in the above example, Simple API also has the function of type conversion. If the parameter you pass in is legal for the declared type, it will be converted to the declared type without manual operation:

```python
class JustTest(View):
    def get(self, request, last_time: datetime.date = Query(...)):
        print(last_time, type(last_time))
        # 2008-08-08 <class 'datetime.date'>
        return HttpResponse(last_time)
```

Use `Query(...)`  to declare the parameter, which means this parameter is required. If there is no `id` parameter in the query string for url, an error will be returned:

```shell script
[
    {
        "loc": [
            "id"
        ],
        "msg": "field required",
        "type": "value_error.missing"
    }
]
```

In addition, you can use default parameters like this:

```python
class JustTest(View):
    def get(self, request, id: int = Query(10)):
        return HttpResponse(id)
    # Or
    def get(self, request, id: int = Query(None)):
        return HttpResponse(id)
```

Or you can use the `default_factory` parameter and pass in a function to dynamically calculate the default value:

```python
def func():
    return 1000

class JustTest(View):
    def get(self, request, id: int = Query(default_factory=func)):
        print(id)  # 1000
        return HttpResponse(id)
```

But you cannot use `default` and `default_factory` at the same time, otherwise an error will be reported:

```shell script
ValueError: cannot specify both default and default_factory
```

In addition to the `default`、`default_factory`, you can also use more attributes to constrain parameters, such as:

```python
class JustTest(View):
    # Use `const` to constrain the parameter value must be the same as the default value
    def get(self, request, param: int = Query(10, const=True)):
        print(param, type(param))
        return HttpResponse(param)

    # If your parameter is of numeric type , you can use `ge`、`gt`、`le`、`lt`、`multipleOf` and other attributes
    def get(self, request,
            param1: int = Query(..., gt=10),  # must be > 10
            param2: int = Query(..., ge=10),  # must be >= 10
            param3: int = Query(..., lt=10),  # must be < 10
            param4: int = Query(..., le=10),  # must be <= 10
            param5: int = Query(..., multipleOf=10),  # must be a multiple of 10
        ):
        return HttpResponse(param)
```

And there are more attributes applied to `str` or `list` type, you can refer to the following table:

#### Field parameter description
| Name           | description |
| ---            | ---         |
| default        | since this is replacing the field’s default, its first argument is used to set the default, use ellipsis (``...``) to indicate the field is required|
| default_factory| callable that will be called when a default value is needed for this field. If both `default` and `default_factory` are set, an error is raised.|
| alias          | the public name of the field|
| title          | can be any string, used in the schema|
| description    | can be any string, used in the schema|
| const          | this field is required and *must* take it's default value|
| gt             | only applies to numbers, requires the field to be "greater than". The schema will have an ``exclusiveMinimum`` validation keyword|
| ge             | only applies to numbers, requires the field to be "greater than or equal to". The schema will have a ``minimum`` validation keyword|
| lt             | only applies to numbers, requires the field to be "less than". The schema will have an ``exclusiveMaximum`` validation keyword|
| le             | only applies to numbers, requires the field to be "less than or equal to". The schema will have a ``maximum`` validation keyword|
| multiple_of    | only applies to numbers, requires the field to be "a multiple of". The schema will have a ``multipleOf`` validation keyword|
| min_items      | only applies to list or tuple and set, requires the field to have a minimum length.|
| max_items      | only applies to list or tuple and set, requires the field to have a maximum length.|
| min_length     | only applies to strings, requires the field to have a minimum length. The schema will have a ``maximum`` validation keyword|
| max_length     | only applies to strings, requires the field to have a maximum length. The schema will have a ``maxLength`` validation keyword|
| regex          | only applies to strings, requires the field match again a regular expression pattern string. The schema will have a ``pattern`` validation keyword|
| extra          | any additional keyword arguments will be added as is to the schema|

When you finish the above tutorial, you can already declare parameters well. If you have registered the `SimpleApiMiddleware` middleware, then all parameters will be automatically verified, and the detailed information of these parameters will be displayed in the interface document. The following will teach you how to generate the interface document.


### Generate documentation
If you want to automatically generate interface documentation, you must add the url of Simple API to your url.py like this:
```python
# url.py
from django.urls import include, path
from django.conf import settings


# Your urls
urlpatterns = [
    ...
]

# Simple API urls, should only run in a test environment.
if settings.DEBUG:
    urlpatterns += [
        # generate documentation
        path(
            "docs/",
            include("django_simple_api.urls"),
            {
                "template_name": "swagger.html",
                "title": "Django Simple API",
                "description": "This is description of your interface document.",
                "version": "0.1.0",
            },
        ),
    ]
```

In the above example, you can modify the `template_name` to change the UI theme of the interface document, We currently have two UI themes: `swagger.html` and `redoc.html`.

And then you can modify `title`、`description` and `version` to describe your interface documentation.

If you are using `class-view`, you can now generate documentation. Start your service, if your service is running locally, you can visit http://127.0.0.1:8000/docs/ to view your documentation.

But if you are using `view-function`, you must declare the request method supported by the view function:

```python
# views.py
from django_simple_api import allow_request_method

@allow_request_method("get")
def just_test(request, id: int = Query(...)):
    return HttpResponse(id)
```

`allow_request_method` can only declare one request method, and it must be in `['get', 'post', 'put', 'patch', 'delete', 'head', 'trace']`.
We do not support the behavior of using multiple request methods in a `view-function`, which will cause trouble for generating documentation.

Note that if you use `@allow_request_method("get")` to declare a request method, you will not be able to use request methods other than `get`, otherwise it will return `405 Method Not Allow`.

You can also not use `allow_request_method`, this will not have any negative effects, but it will not generate documentation.
We will use `warning.warn()` to remind you, this is not a problem, just to prevent you from forgetting to use it.

Now, the view function can also generate documents, you can continue to visit your server to view the effect.
