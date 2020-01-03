# httpRequest ->DRF Request ->  APIView -> genericApiView + Mixins 的总结
这个总结是为了加深对django rest_framework的理解，通过代码一步一步的迭代，改进来理解使用框架的便利。

## DRF: Request&Response 代替了 Django: HttpRequest&SimpleTemplateResponse
DRF中引入了`Request`和 `Response` 对象进行请求和响应，代替了Django中的`HttpRequest`和`SimpleTemplateResponse`
1. Request
1.1 解析请求（Request Parsing）
DRF默认使用了``JSONParser`类进行解析，因此我们在返回JSON数据时不需要做任何工作。

1.2 身份验证（Authentication）
DRF中提供了按请求进行认证的机制。
- 对API的不同部分使用不同的认证策略
- 支持多种身份验证策略
- 对每个请求了用户和token信息

`Request`中和身份验证相关的几个属性
### request.user
该属性返回请求的用户，如果请求经过身份验证，则返回`django.contrib.auth.models.User`; 
如果未经过身份验证，则返回`django.contrib.auth.models.AnonymousUser`.

### request.auth
该属性返回身份验证的上下文，它取决于所使用的身份验证策略，但一般在`request.auth`中带有Token，
如果请求未经过身份验证，则该值为None。

2. Response
Response类继承了Django的`SimpleTemplateResponse`，相比于`HttpResponse`和它的父类，Response只是提供了一个更好的界面。
同时，和`HttpResponse`对象相比，Response不需要通过实力化`Render`进行渲染，而是直接传入为渲染的数据。
由于`Response`的渲染器无法处理复杂类型的数据，如Django中的Model实例，因此，需要在响应之前通过`Serializer`类进行序列化操作，
将复杂类型序列化为Python原始的数据类型。

`Response`使用时，需要进行导入
```
from rest_framework.response import Response
```
其构造方法是：
```
Response(data, status=None)
```
data是响应给客户端的序列化后的数据
status是响应状态码，默认为200

## APIView
DRF中APIView继承了Django的View， 相比起来有以下特点
- 传递给处理方法的请求是DRF的`Request`实例，而不是Django的`HttpRequest`实例
- 响应并返回的是DRF的`Response`对象，而不是Django的`HttpResponse`对象
- 任何APIException异常都会被捕获并调制到适当的响应中
- 会对接手的请求进行身份认证和权限的检查

和View类似，使用APIView时，接收到的请求会被dispatch到对应的方法中，比如get(), post()等
```python
from rest_framework.views import APIView

class SnippetList(APIView):
    # 処理GETリクエスト
    def get(self, request, format=None):
        snippet = Snippet.objects.all()
        serializer = SinppetSerializer(snippet, many=True)
        return Response(serializer.data)
    
    # 処理POSTリクエスト
    def post(self, request, format=None):
        serialiezer = SnippetSerializer(data=request.data)
        if serializer.is_valid():
            serialiezer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serialiezer.data, status=status.HTTP_400_BAD_REQUEST)
```

## GenericAPIView
`GenericAPIView`继承于APIView，为常用的列表视图和详细视图提供了一些操作属性方法。
```python
views.py

from rest_framework import generics
from .serializers import SnippetSerializer
from .models import Snippet

class Show(generics.GenericAPIView):
    # 指定序列化类
    serializer_class = SnippetSerializer

    # 指定QuerySet对象
    queryset = Snippet.objects.all()

    def get(self, request):
        return Response("hello 2020!")

    def post(self, request):
        return Response(request.data)
```

## Mixin
通常GenericAPIView不会单独使用，因为它相比APIView，仅仅提供了一些公共的用于列表视图和详细视图的属性和方法。
GenericAPIView需要和Mixins组合使用。

Mixin类中提供了许多用于对视图进行操作的方法，这些操作方法无需我们自己实现就可以实现，比如，`ListModelMixin`提供了一个`list()`方法
用于将Model的查询集响应给客户端，因此可以用于列表视图的`get()`中：
```python
class show(generics.GenericAPIView, mixins.ListModelMixin):

    serializer_class = SnippetSerializer
    queryset = Snippet.objects.all()

    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)
```
这个`ListModelMixin`的list(request, *args, **kwargs) 方法用于列出查询集QuerySet。
如果queryset被填充，则响应码为200， 并将查询集序列化作为响应体响应给客户端。

### CreateModelMixin
该类提供了一个`.create(request, *args, **kwargs)`方法，用于创建并保存一个新的Model实例，因此用在POST请求中，和`post()`方法组合使用。
如果Model实例创建成功，则响应码为201 Created， 并将对象序列化作为响应体响应给客户端；如果Model实例创建失败，则响应码为401(Bad Request)
并将错误信息作为响应体。

除此以外，比如还有 
### RetrieveModelMixin
用于检索并返回一个想有的Model实例

### UpdateModelMixin
用于更新并保存现有的Model对象。

### DestroyModelMixin
顾名思义，用于删除一个已存在的Model实例。
```python
class SnippetDetail(mixins.RetrieveModelMixin,
                    mixins.UpdateModelMixin,
                    mixins.DestroyModelMixin,
                    mixins.CreateModelMixin,
                    generics.GenericAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer
    # get，put,delete方法是GenericAPIView中提供
    # retrieve,update,destory方法由Mixin类视图提供
    # 二者可以灵活的结合
    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

    def put(self, request, *args, **kwargs):
        return self.update(request, *args, **kwargs)

    def delete(self, request, *args, **kwargs):
        return self.destroy(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)
```
:バツ: 说实话，其实没有特别搞懂，这块的优点，没理解必须要用Mixin的理由。
他说的上面的方法我们都不需要再自己进行实现，但是我感觉GenericViewAPI已经都实现了啊。

## 常用具体通用视图
在DRF中，除了提供`Mixin`类之外，还提供了`GenericAPIView`和`Mixins`的组合View类，这些类称为`具体通用视图`。 
因此我们只需要具体通用视图就可以实现我们所需要的功能。比如：
```python
from rest_framework import generics

class show(generics.ListCreateAPIView):
    serializer_class = SnippetSerializer
    queryset = Snippet.objects.all()
```
:鳥居: 在这个示例中，`ListCreateAPIView`继承了GenericAPIView, ListModelMixin, CreateModelMixin这3个类，
并且内部都已经将对应请求方法和操作方法进行了实现，我们只需要制定一些属性即可，无需在写`get()`,`post()`请求方法逻辑。

### ListCreateAPIView
父类： `GenericAPIView`, `ListModelMixin`, `CreateModelMixin`
特点：已经提供了 `get()`, `put()`方法处理

### RetrieveUpdateDestroyAPIView
父类： `GenericAPIView`, `RetrieveModelMixin`, `UpdateModelMixin`, `DestroyModelMixin`
特点：已经提供了`get()`, `put()`, `patch()`, `destory()`方法处理

## 重构View示例
现在，利用以上View内容以及Django中的函数视图等内容，从函数视图开始，一步步对视图进行重构，看看其简化的过程。
### Step1 最初的函数视图
```python
# 列表视图
@api_view(['GET', 'POST'])
def student_list(request):

    # 处理GET请求
    if request.method == 'GET':

        student = Student.objects.all()
        serializers = StudentSerializer(student, many=True)
        return Response(serializers.data)

    # 处理POST请求
    if request.method == 'POST':
        serializer = StudentSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(status=status.HTTP_400_BAD_REQUEST)


# 详情视图
@api_view(['GET', 'POST'])
def student_detail(request, pk):
    if request.method == 'GET':

        student = Student.objects.get(pk=pk)
        serializer = StudentSerializer(student)
        return Response(serializer.data)

    if request.method == 'POST':

        serializer = StudentSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(status=status.HTTP_400_BAD_REQUEST)
```

### Step2 将函数视图用APIView替换
现在通过APIView进行重构，可以定义请求对应的`get()`方法和`post()`方法。
```python
from rest_framework.response import Response
from hello.models import Student
from hello.serializers import StudentSerializer
from rest_framework import status
from rest_framework import views

# 列表视图
class StudentList(views.APIView):

    # GET请求
    def get(self,request):
        student = Student.objects.all()
        serializer = StudentSerializer(student,many=True)
        return Response(serializer.data)

    # POST请求
    def post(self, request):

        serializer = StudentSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.data, status=status.HTTP_400_BAD_REQUEST)


# 详情视图
class StudentDetail(views.APIView):

    def get(self, request, pk):
        student = Student.objects.get(pk=pk)
        serializer = StudentSerializer(student)
        return Response(serializer.data)

    def post(self, request):
        serializer = StudentSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.data, status=status.HTTP_400_BAD_REQUEST)
```

### Step3 GenericAPIView+Mixins替换APIView
GenericAPIView相比，APIView，提供了部分属性，Mixins提供了许多用于对视图进行操作的方法
```python
from rest_framework.response import Response
from hello.models import Student
from hello.serializers import StudentSerializer
from rest_framework import status
from rest_framework import generics
from rest_framework import mixins
from django.http import Http404


class StudentList(generics.GenericAPIView, minxins.ListModelMixin, mixins.CreateModelMixin):

    # 指定序列化类
    serializer_class = StudentSerializer
    # 执行Model对象查询集
    queryset = Student.objects.all()
    # 如果不对query_set的值进行修改，没有必要重写该方法
    def get_queryset(self):
        return Student.objects.all()

    def get(self, request):
        return self.list(request)

    def post(self, request):
        return self.create(request)

# 详情视图
class StudentDetail(generics.GenericAPIView,
                    mixins.RetrieveModelMixin):

    def get(self, request, *args, **kwargs):
        return self.retrieve(self, request, *args, **kwargs)
```

### Step4 使用具体通用视图替换GenericAPIView+Mixins
```python
from hello.models import Student
from hello.serializers import StudentSerializer
from rest_framework import generics

# 列表视图
class StudentList(generics.ListCreateAPIView):

    serializer_class = StudentSerializer
    queryset = Student.objects.all()


# 详情视图
class StudentDetail(generics.RetrieveUpdateDestroyAPIView):

    queryset = Student.objects.all()
    serializer_class = StudentSerializer
```
这个真是有点便利了，请求的对应方法被直接继承到了2个类里面。

DRF中提供了这么多View， 但无非就是从views.View这个基类，通过继承，mixin实现而已。
因此，在学习这些类是，不必死记硬背，而是通过其继承结构，实现源码，了解明白它的作用和使用方式，以及和mixin的混合使用。


# ViewSets和Routers
接下来我们来说一下，ViewSets和Routers。
在DRF中，允许在一个类中组合相关视图的逻辑，称为`ViewSets`。
比如通过通用视图，可以定义列表视图，详细视图等等，但每个视图位于不同的类中，而通过ViewSets则可以多个视图放在同一个类中。

:鳥居: ViewSets也是一种基于View类的视图，只不过和APIView不同的是，它并不提供如`get()`, `post()`等和HTTP请求相对应的方法，而提供是如
`list()`, `create()`这样的操作方法。

在配置ViewSets的URL时，一般不会显示进行配置，而是使用`Routers`类来注册ViewSets，Routers会自动确定URL格式。 
`Router`可以将请求和视图自动进行匹配，并应设相应的处理逻辑。

如果我们使用的是ViewSets而非View，那么我们就没有必要自己去配置每个View对应的URL了，直接使用Router注册一个视图集，让Router来完成剩下的工作。

## viewsets类
所有的viewsets类都是直接或者间接的继承与`ViewSetMixin`这个基类。目前viewsets相关类包中有4个类。
### ViewSet
ViewSet继承了ViewSetMixin和APIView

### GenericViewSet
GenericViewSet继承了`ViewSetMixin`和`GenericAPIView`，因此具有GenericAPIView拥有的一些属性和方法。

### ModelViewSet
该类继承了`GenericViewSet`，并通过混合各种`Mixin`类的行为，实现了各种动作的实现。该类的源码如下：
```python
class ModelViewSet(mixins.CreateModelMixin,
                   mixins.RetrieveModelMixin,
                   mixins.UpdateModelMixin,
                   mixins.DestroyModelMixin,
                   mixins.ListModelMixin,
                   GenericViewSet):
    """
    A viewset that provides default `create()`, `retrieve()`, `update()`,
    `partial_update()`, `destroy()` and `list()` actions.
    """
    pass
```
:鳥居: 因此，通过一个`ModelViewSet`就可以完成包含列表视图，详细视图等多个视图的请求的操作！！

下面开始总结如何通过Router给ViewSet配置URL。
## Router
到现在为止我们对一个View配置URL是通过如下方式进行的：
```python
urlpatterns = [
    path('show/', views.ShowList.as_view()),  # 用于列表视图
    path('show/<pk>', views.ShowDetail.as_view())  #用于详情视图
]
```
可以看出，这种方式有些麻烦，针对于不同的View，需要单独的进行配置。
如果是通过ViewSets实现的View，那么通过Router就可以自动的配置URL了，我们只需要给Router注册一个ViewSets，其他的什么都不用管。

```python
from rest_framework import routers

from rest_framework.routers import DefaultRouter

router = DefaultRouter()
# 给router注册ShowViewSet
router.register('show', views.ShowViewSet, base_name='show')

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include(router.urls)),
]
```
其中，`register()`方法用来注册一个ViewSets，其所需要的参数如下；
- `prefix`： 用于这组路由的URL前缀，该参数必须设定
- `viewset`： ViewSet类，该参数必须设定
- `base_name`： 该参数可选，用于指定创建URL的名称，如果未指定，则默认使用viewsets中的queryset属性自动生成，因此，如果viewsets中没有声明`queryset`属性，则该参数必须设置。