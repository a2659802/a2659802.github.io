---
layout: post
title: 记一次排查django后台返回DateTimeField时区错误的问题
date: 2023-02-23
categories: blog
tags: [django]
description: 排查django rest framework时区错误问题
---

# 记一次排查django后台返回DateTimeField时区错误的问题
## 背景
因工作原因接手了一个由老版django开发的web项目, 其采用django+vue前后端分离架构。
然而在django通过json返回的时间数据中，时间总是相差8小时。 

注意：本文针对的情况为特定版本, 在最新版中是没有这个问题的

```
django                  1.8.3
djangorestframework     3.6.2
```

## 过程

看到8这个数字基本都能判断是时区错误导致, 一开始以为是django的问题, 仔细阅读官网相关的配置
```python
# 项目的部分配置
LANGUAGE_CODE = 'zh-hans'
TIME_ZONE = 'Asia/Shanghai'
USE_I18N = True
```
发现时区配置是没问题的, 而且某个和前端无交互的接口中用到了相同的数据写到文件中, 输出的文件内容时区没有问题.

### 临时重写序列化类应急

那么问题肯定出在对象序列化成json过程. 由于第二天还有项目演示的, 花了不多的时间自己重写了一个序列化类
```python
from rest_framework import serializers
# 继承并重写to_representation方法(这里就是将对象序列化成字符串的函数)
class TimeWithTimezoneField(serializers.DateTimeField):

    default_error_messages = {
        'invalid': 'Time has wrong format, expecting %H:%M:%S%z.',
    }

    def __init__(self, *args, **kwargs):
        super(serializers.DateTimeField,self).__init__(*args, **kwargs)

    def to_representation(self, value):
        if not value:
            return None

        if isinstance(value, str):
            return value

        return timezone.make_naive(value).strftime("%Y-%m-%d %H:%M:%S")

```
然后将`update_time = serializers.DateTimeField()`改为`update_time = TimeWithTimezoneField()`, 时区错误的问题解决

### 真正解决
上面的方法最大的问题是这个项目已经有大量引用DateTimeField的代码, 将它们一一修改是不可能的
由于对django不熟悉, 花了点时间去照着官网的教程搓了一个demo出来, 发现时区问题并不是django框架导致的,
而是一个叫DjangoRestFramework(DRF)的框架来处理的. 于是又花了点时间研究DRF, 走了点弯路

分别用新版Django+新版DRF搭了个demo, 然后将其移植到旧版django+旧版DRF中。
运行后发现两者对于时间的数据，新版能正确处理，旧版则不行。

这时我已经九成把握确定是旧版框架bug, 于是打开github，找到对应的代码进行对比

```python
# 3.6.2代码 https://github.com/encode/django-rest-framework/blob/928f7cb40fa6bb21b887d53577e4c00cf7110661/rest_framework/fields.py#L1100-L1159
class DateTimeField(Field):
    def enforce_timezone(self, value):
        """
        When `self.default_timezone` is `None`, always return naive datetimes.
        When `self.default_timezone` is not `None`, always return aware datetimes.
        """
        field_timezone = getattr(self, 'timezone', self.default_timezone())

        if (field_timezone is not None) and not timezone.is_aware(value):
            return timezone.make_aware(value, field_timezone)
        elif (field_timezone is None) and timezone.is_aware(value):
            return timezone.make_naive(value, utc)
        return value

    def default_timezone(self):
        return timezone.get_default_timezone() if settings.USE_TZ else None

    def to_representation(self, value):
        if not value:
            return None

        output_format = getattr(self, 'format', api_settings.DATETIME_FORMAT)

        if output_format is None or isinstance(value, six.string_types):
            return value

        if output_format.lower() == ISO_8601:
            value = value.isoformat()
            if value.endswith('+00:00'):
                value = value[:-6] + 'Z'
            return value
        return value.strftime(output_format)
```

```python
#master分支代码 https://github.com/encode/django-rest-framework/blob/15c613a9eb645c63102b9e894199bcf1c9bf4d65/rest_framework/fields.py#L1143-L1205
class DateTimeField(Field):
    def enforce_timezone(self, value):
        """
        When `self.default_timezone` is `None`, always return naive datetimes.
        When `self.default_timezone` is not `None`, always return aware datetimes.
        """
        field_timezone = self.timezone if hasattr(self, 'timezone') else self.default_timezone()

        if field_timezone is not None:
            if timezone.is_aware(value):
                try:
                    return value.astimezone(field_timezone)
                except OverflowError:
                    self.fail('overflow')
            try:
                return timezone.make_aware(value, field_timezone)
            except InvalidTimeError:
                self.fail('make_aware', timezone=field_timezone)
        elif (field_timezone is None) and timezone.is_aware(value):
            return timezone.make_naive(value, datetime.timezone.utc)
        return value

    def default_timezone(self):
        return timezone.get_current_timezone() if settings.USE_TZ else None

    def to_representation(self, value):
        if not value:
            return None

        output_format = getattr(self, 'format', api_settings.DATETIME_FORMAT)

        if output_format is None or isinstance(value, str):
            return value

        value = self.enforce_timezone(value)

        if output_format.lower() == ISO_8601:
            value = value.isoformat()
            if value.endswith('+00:00'):
                value = value[:-6] + 'Z'
            return value
        return value.strftime(output_format)
```

可以看到旧版的enforce_timezone没有在序列化时调用, 而且enforce_timezone的实现也不同。
将新版的这部分代码拷贝, 覆盖旧版代码. 然后运行demo项目，时区问题解决！

## 后记

后来翻了以下commit, 应该是在[这个提交](https://github.com/encode/django-rest-framework/commit/da535d31dd93dbb1d650e2e92bd0910ca8eb4ea4)得到了修复

也就是说其实我可以将3.6.2升级到3.6.5来解决. 但是想想还是算了，万一升了又有别的bug就不好了