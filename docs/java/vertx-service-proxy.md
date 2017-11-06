# Vert.x 3 服务代理

> 服务代理提供了在事件总线上暴露服务, 并且减少调用服务所要求的代码的一种方式. 服务代理帮助你解析事件总线的消息结构和接口方法的映射关系, 比如方法名称, 参数等等. 本质上是一种RPC. 它通过代码生成的方式创建服务代理.

因此需要实现两个Verticle, 一个为 `DatabaseService`, 提供数据库操作服务. 另一个 Verticle 用于请求数据库操作.

Verticle 运行在一个两节点的集群环境中中

## 提供服务

要提供一个服务, 需要一个服务接口, 一个实现和一个Verticle

- 服务接口: `DatabaseService`, 服务接口, 定义了数据操作接口和代理创建接口.
- 服务实现: `DatabaseServiceImpl`, 服务实现类, 实际的数据库操作在这里实现
- 服务注册: `DatabaseServiceVerticle`, 用于注册服务

服务接口是通过 `@ProxyGen` 标注的接口, 例如:

```
package com.totorotec.service;

import io.vertx.codegen.annotations.ProxyGen;
import io.vertx.codegen.annotations.VertxGen;
import io.vertx.core.AsyncResult;
import io.vertx.core.Handler;
import io.vertx.core.Vertx;
import io.vertx.core.json.JsonObject;

import com.totorotec.service.impl.DatabaseServiceImpl;
@ProxyGen
@VertxGen
public interface DatabaseService {
  // 一些用于创建服务实例和服务代理实例的工厂方法
  static DatabaseService create(Vertx vertx) {
    return new DatabaseServiceImpl(vertx);
  }
  static DatabaseService createProxy(Vertx vertx, String address) {
    return new DatabaseServiceVertxEBProxy(vertx, address);
  }
  // 实际的服务方法
  // void save(String collection, JsonObject document, Handler<AsyncResult<Void>> resultHandler);
  public void getUserById(int id, Handler<AsyncResult<JsonObject>> resultHandler);
}
```

> 注意: `java.lang.IllegalStateException: Cannot find proxyClass`, 把`maven-compiler-plugin`插件的版本升级到`3.7.0`

```
<plugin>
  <artifactId>maven-compiler-plugin</artifactId>
  <version>3.7.0</version>
  <configuration>
    <source>1.8</source>
    <target>1.8</target>
  </configuration>
</plugin>
```

> 另外: 需要给插件 `maven-compiler-plugin` 增加符号处理器配置 `annotationProcessor`, 如下

```
<plugin>
  <artifactId>maven-compiler-plugin</artifactId>
  <version>3.7.0</version>
  <configuration>
    <source>1.8</source>
    <target>1.8</target>
    <annotationProcessors>
      <annotationProcessor>io.vertx.codegen.CodeGenProcessor</annotationProcessor>
    </annotationProcessors>
  </configuration>
</plugin>
```


## 代理创建

代理创建有两种方式, 分别是:

- ServiceProxyBuilder
- VertxEBProxy

第一种是手工的通过 ServiceProxyBuilder 类构造一个代理类, 第二种是通过生成的代码来创建代理, 下面我们分别说明两种方式是如何创建代理类的.

### 第一种, 使用 ServiceProxyBuilder

```
ServiceProxyBuilder builder = new ServiceProxyBuilder(vertx).setAddress(DatabaseServiceVerticle.SERVICE_ADDRESS);
// 构造服务
DatabaseService service1 = builder.build(DatabaseService.class);
```

实例化 ServiceProxyBuilder 类, 需要传入的参数为:

- Vert.x 实例对象,
- 服务在事件总线上的地址

让后调用 **ServiceProxyBuilder** 实例对象的 **build** 方法, 并传入 服务接口类CLASS.

### 第二种, 使用生成的代码创建

首先要在接口中添加一个静态方法 **createProxy**, 用于实例化 **${ServiceInterfaceName}VertxEBProxy** 代理类

```
public interface DatabaseService {
  ...
  static DatabaseService create(Vertx vertx) {
    return new DatabaseServiceImpl(vertx);
  }
  static DatabaseService createProxy(Vertx vertx, String address) {
    return new DatabaseServiceVertxEBProxy(vertx, address);
  }
  ...
}
```