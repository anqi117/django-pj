# httpRequest -> APIView -> genericApiView + Mixins 的总结
这个总结是为了加深对django rest_framework的理解，通过代码一步一步的迭代，改进来理解使用框架的便利。

## DRF中引入了`Request`和 `Response` 对象进行请求和相应，代替了Django中的`HttpRequest`和`SimpleTemplateResponse`
