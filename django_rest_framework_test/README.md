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