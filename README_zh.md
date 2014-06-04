
# HTTP API 设计指南

## 介绍

该指南内容脱胎于[Heroku Platform API](https://devcenter.heroku.com/articles/platform-api-reference)的设计工作，描述了一套用于设计HTTP+JSON API的实践规范。

相比原内容，该篇文章做出了扩充并加入了Heroku内部APIs的设计指南，希望Heroku之外的API设计者感兴趣。

我们的目标是保持一致性，专注于业务逻辑而避免纠缠于细枝末节的问题。我们追寻的是一个_良好的，一致的，有据可依的方法_来设计API，当然不一定是_唯一的理想_的方式。

该指南假定你对于设计HTTP+JSON APIs的一些基础知识有所了解，因而并不覆盖这些基本内容。

欢迎对该指南提出修正的[贡献者](CONTRIBUTING.md) 。

## 内容

*  [返回合适的状态码](#return-appropriate-status-codes)
*  [尽可能的提供全量资源内容](#provide-full-resources-where-available)
*  [请求体中可包含JSON](#accept-serialized-json-in-request-bodies)
*  [为资源提供(UU)IDs](#provide-resource-uuids)
*  [使用标准的时间戳](#provide-standard-timestamps)
*  [使用ISO8601格式的UTC时间](#use-utc-times-formatted-in-iso8601)
*  [使用一致的路径格式](#use-consistent-path-formats)
*  [小写的路径和属性名](#downcase-paths-and-attributes)
*  [嵌入外键关联对象](#nest-foreign-key-relations)
*  [支持非ID的资源标识](#support-non-id-dereferencing-for-convenience)
*  [生成结构化的错误信息](#generate-structured-errors)
*  [支持Etags缓存](#support-caching-with-etags)
*  [用Request-Ids追踪请求](#trace-requests-with-request-ids)
*  [使用ranges处理翻页](#paginate-with-ranges)
*  [显示接口访问频次的信息](#show-rate-limit-status)
*  [在Accepts header中指定版本](#version-with-accepts-header)
*  [简化嵌套路径](#minimize-path-nesting)
*  [提供机器可识别的JSON schema](#provide-machine-readable-json-schema)
*  [提供用户友好的文档](#provide-human-readable-docs)
*  [提供可执行的示例](#provide-executable-examples)
*  [描述稳定性](#describe-stability)
*  [使用SSL](#require-ssl)
*  [默认返回可直观阅读的JSON格式](#pretty-print-json-by-default)

### <a id="return-appropriate-status-codes"></a>返回合适的状态码
每次请求都应该返回合适的状态码，成功的响应应该参照以下规则：

* `200`: `GET`请求成功后返回， 同样针对同步完成的`DELETE`或
  `PATCH`请求
* `201`: 对于同步成功完成的 `POST` 请求
* `202`: `POST`， `DELETE`或 `PATCH` 请求已被接受，但需要异步处理
* `206`: `GET`请求已成功，但返回的是部分结果，参见 [ranges](#paginate-with-ranges)

针对状态码，用户错误和服务器错误信息，请参考 [HTTP response code spec](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html)。

### <a id="provide-full-resources-where-available"></a>尽可能的提供全量资源内容

API的响应中应尽可能的提供全量资源内容（例如返回的对象要包含所有属性值），对于返回状态码是200和201的响应，必须返回全量资源，包括`PUT`/`PATCH` 和 `DELETE`请求，如：

```
$ curl -X DELETE \  
  https://service.com/apps/1f9b/domains/0fd4

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
...
{
  "created_at": "2012-01-01T12:00:00Z",
  "hostname": "subdomain.example.com",
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "updated_at": "2012-01-01T12:00:00Z"
}
```

状态码是202的响应不需要包含全量资源，如：

```
$ curl -X DELETE \  
  https://service.com/apps/1f9b/dynos/05bd

HTTP/1.1 202 Accepted
Content-Type: application/json;charset=utf-8
...
{}
```

### <a id="accept-serialized-json-in-request-bodies"></a>请求体中可包含JSON

可接受在请求体中包含JSON的`PUT`/`PATCH`/`POST`请求，JSON数据可以是form-encoded的，也可以独立存在的，这为包含JSON的响应提供了对称性，如：
```
$ curl -X POST https://service.com/apps \
    -H "Content-Type: application/json" \
    -d '{"name": "demoapp"}'

{
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "name": "demoapp",
  "owner": {
    "email": "username@example.com",
    "id": "01234567-89ab-cdef-0123-456789abcdef"
  },
  ...
}
```

### <a id="provide-resource-uuids"></a>为资源提供(UU)IDs

默认为每个资源提供一个`id`的属性。如果没有其他原因，尽量使用UUID。
不要使用在你的Service中全局不唯一的ID，特别不要使用自增型的ID。

UUID转化为小写的， `8-4-4-4-12` 这样的格式， 如：
```
"id": "01234567-89ab-cdef-0123-456789abcdef"
```

### <a id="provide-standard-timestamps"></a>使用标准的时间戳

默认为资源提供 `created_at` 和 `updated_at` 两个时间戳属性，
如:

```json
{
  ...
  "created_at": "2012-01-01T12:00:00Z",
  "updated_at": "2012-01-01T13:00:00Z",
  ...
}
```

对于一些资源这两个时间戳可能没有意义，因此可以省去。

### <a id="use-utc-times-formatted-in-iso8601"></a>使用ISO8601格式的UTC时间

只接受和返回UTC时间，并格式化为ISO8601的格式，如：

```
"finished_at": "2012-01-01T12:00:00Z"
```

### <a id="use-consistent-path-formats"></a>使用一致的路径格式

#### 资源名

使用复数形式的资源名除非该资源在系统中为单例的（例如在大多数系统中，每个用户对应只有一个account的资源）。这种规则能保证你指向某个资源的方式一致。

#### 操作

设计资源地址（endpoint layouts）时应该尽量避免对单一资源有特殊的操作（指HTTP 动作之外的操作），当无法避免的时候，加上一个`actions`的前缀来描述：
```
/resources/:resource/actions/:action
```

如：

```
/runs/{run_id}/actions/stop
```

### <a id="downcase-paths-and-attributes"></a>小写的路径和属性名

使用小写的，横线分割的路径名，如：

```
service-api.com/users
service-api.com/app-setups
```

属性名也要小写，但使用下划线来分割，这样在Javascript中属性名就不需要再加上引号就直接成为了类型。

```
"service_class": "first"
```

### <a id="nest-foreign-key-relations"></a>嵌入外键关联对象

如果有外键，应该作为一个嵌入对象：

```json
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0..."
  },
  ...
}
```
  
而不是

```json
{
  "name": "service-production",
  "owner_id": "5d8201b0...",
  ...
}
```

这样做当扩充关联的对象时，结构不需要变化，如：

```json
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0...",
    "name": "Alice",
    "email": "alice@heroku.com"
  },
  ...
}
```

### <a id="support-non-id-dereferencing-for-convenience"></a>支持非ID的资源标识

一些情况下让用户使用ID来标识资源显得不那么方便，比如在Heroku中，用户也许更熟悉app name，尽管UUID才是系统用的标识。在这种情况下，id和name都能被识别为该资源也许是个不错的主意：
```
$ curl https://service.com/apps/{app_id_or_name}
$ curl https://service.com/apps/97addcf0-c182
$ curl https://service.com/apps/www-prod
```

当然不能只接受app name，id同样需要。

### <a id="generate-structured-errors"></a>生成结构化的错误信息

当发生错误时，响应体应该包含一致的结构化的错误信息。其中应包含一个可机器识别的错误 `id`，可读的`message`和一个可选的字段`url`链接到更多的错误信息及可能的解决办法。如：

```
HTTP/1.1 429 Too Many Requests
```

```json
{
  "id":      "rate_limit",
  "message": "Account reached its API rate limit.",
  "url":     "https://docs.service.com/rate-limits"
}
```

错误信息格式和用户可能遇到的错误标识`id`都应该被文档化。


### <a id="support-caching-with-etags"></a>支持Etags缓存

所有响应头信息中应包括`ETag`，识别特定的资源版本。用户在接下来的请求中可以用`If-None-Match`请求头来检测该版本是否过期。

### <a id="trace-requests-with-request-ids"></a>用Request-Ids追踪请求

在每个API请求响应中包含`Request-Id`，采用UUID，如果服务器和客户端都记录了该值，可以非常方便的定位调试请求。

### <a id="paginate-with-ranges"></a>使用ranges处理翻页

当响应结果包含大量数据的时候应该使用翻页来处理。使用头信息`Content-Range`来表达支持翻页的请求。参见实例 [Heroku Platform API on Ranges](https://devcenter.heroku.com/articles/platform-api-reference#ranges)来了解更多关于请求响应头，状态码，限制结果数，排序和页面定位等信息。

### <a id="show-rate-limit-status"></a>显示接口访问频次的信息

限制客户端的调用次数能够保持服务的健康度，保证其他用户的服务质量。可以参考[token bucket algorithm](http://en.wikipedia.org/wiki/Token_bucket)去量化请求次数的限制。

在响应头信息中包含`RateLimit-Remaining`来告诉客户端该request token可用的调用次数。

### <a id="version-with-accepts-header"></a>在Accepts header中指定版本

从伊始就应该指定API的版本。客户端使用`Accepts` header加上自定义的content type去和服务端沟通所要消费的API版本。如:
```
Accept: application/vnd.heroku+json; version=3
```

最好不要使用默认版本号，让用户显式的指定。

### <a id="minimize-path-nesting"></a>简化嵌套路径

父子关系的资源数据往往需要嵌套路径来表示，有些嵌套层级可能会很深，如：
```
/orgs/{org_id}/apps/{app_id}/dynos/{dyno_id}
```

应该尽量的减少嵌套层级，使资源直接暴露于根路径。使用嵌套来标识scoped collections：
```
/orgs/{org_id}
/orgs/{org_id}/apps
/apps/{app_id}
/apps/{app_id}/dynos
/dynos/{dyno_id}
```

### <a id="provide-machine-readable-json-schema"></a>提供机器可识别的JSON schema

提供机器可识别的JSON schema来准确定义你的API。使用工具[prmd](https://github.com/interagent/prmd)来管理schema。执行`prmd verify`来确保你的schema合法。

### <a id="provide-human-readable-docs"></a>提供用户友好的文档

提供用户友好的文档，方便开发者理解你的API。 

如果你使用了上文提到的prmd来定义你的schema，那么你可以直接运行`prmd doc`来生成所有endpoint的Markdown格式的文档。

除了endpoint的信息之外, 提供一个包含以下信息的API概括：

* 认证， 如何获取和使用authentication tokens。
* API稳定性和版本，包含怎么选择使用合适的版本。
* 基本的请求/响应头信息。
* 错误格式。
* 不同语言调用API的示例。

### <a id="provide-executable-examples"></a>提供可执行的示例

提供给用户可以直接在终端中测试的示例。这些示例应该是逐条按正确的顺序给出，为想尝试API调用的用户提供最大的方便。如：
```
$ export TOKEN=... # acquire from dashboard
$ curl -is https://$TOKEN@service.com/users
```

如果你使用了[prmd](https://github.com/interagent/prmd) 来生成Markdown
格式的文档，它会帮你自动生成这些示例。

### <a id="describe-stability"></a>描述稳定性

根据不同的endpoint的成熟度，用prototype/development/production等标识来描述其稳定性。

从[Heroku API compatibility policy](https://devcenter.heroku.com/articles/api-compatibility-policy)来参考如何描述稳定性以及不同稳定度的更改策略。

一旦你的API宣布上线和稳定，不要在同一个版本号下做一些不向后兼容的改变。

### <a id="require-ssl"></a>使用SSL

所有API调用都要通过SSL加密。别问什么时候应该什么时候不应该用，用就是了。。。

### <a id="pretty-print-json-by-default"></a>默认返回可直观阅读的JSON格式

用户对你API的第一印象很有可能来自于命令行下的curl调用，所以如果返回的结果是一个可直观阅读的JSON格式对于用户理解非常有帮助。

```json
{
  "beta": false,
  "email": "alice@heroku.com",
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "last_login": "2012-01-01T12:00:00Z",
  "created_at": "2012-01-01T12:00:00Z",
  "updated_at": "2012-01-01T12:00:00Z"
}
```

而不是：

```json
{"beta":false,"email":"alice@heroku.com","id":"01234567-89ab-cdef-0123-456789abcdef","last_login":"2012-01-01T12:00:00Z", "created_at":"2012-01-01T12:00:00Z","updated_at":"2012-01-01T12:00:00Z"}
```

确保在最后一行包含个换行符，方便用户的终端中使用。

对于大多数性能不敏感的API，可直接阅读格式的响应结果是OK的。对于一些性能敏感的API，可以考虑关闭或者对于某些客户端关闭这样的格式。
