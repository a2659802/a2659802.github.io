---
layout: post
title: django shell调试接口
date: 2023-05-18
categories: blog
tags: [django]
description: 如何用django manage.py shell来调试接口函数
---

# django shell调试接口

## 背景
工作的项目用的是django+django rest framework搭的，一个请求经过多个项目层层转发，效率非常低。表现为调试起来特别慢。所以打算寻找某种办法，直接在终端调试接口

## 方案
如果是普通的函数，其实在终端里面导入，然后传递参数调用即可。对于接口函数有些不同，它有一个request参数，例如：
```python
class AssetViewSet(viewsets.ModelViewSet):
    queryset = alert_models.Assets.objects.all()

    @list_route(methods=['get'], url_path='get_policys_by_asset')
    def get_policys_by_asset(self, request, *args, **kwargs):
        host_id = request.query_params.get("host_id")
        service_id = request.query_params.get("service_id")
```
所以重点是怎么构造request参数，直接看实操

```bash
>>> from pkgname.alert.views import AssetViewSet
>>> view = AssetViewSet()

>>> from rest_framework.test import APIRequestFactory
>>> from rest_framework.request import Request
>>> factory = APIRequestFactory()

>>> _request = factory.get('/api/alarm/get-policy', {'service_ew_id': 2})
>>> request = Request(_request)

>>> resp = view.get_policys_by_asset(request)
>>> resp.data
[OrderedDict([省略响应内容
```

如果想要在不改变代码的情况下，执行单步调试，可以将函数封装到另一个函数，然后用pdb调试接口，例如:

```bash
>>> def trace():
...     import pdb
...     pdb.set_trace()
...     resp = view.get_policys_by_asset(request)
...
>>> trace()
> <console>(4)trace()
(Pdb) s
--Call--
> /usr/share/pkgname/pkgname/alert/views.py(145)get_policys_by_asset()
-> @list_route(methods=['get'], url_path='get_policys_by_asset')
```

## 总结
先通过APIRequestFactory来构造请求，然后转为Request类即可。这里的例子用的是GET请求，POST请求应该也是类似是思路，这里就不多写了。