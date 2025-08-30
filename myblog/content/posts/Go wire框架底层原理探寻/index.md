
+++

date = '2025-08-30T20:56:01+08:00'
draft = false
title = 'Golang Wire框架 底层原理探寻'
summary = '在我从Java转向Go后，一直迷惑于Wire框架的作用，看起来它和Spring框架功能相似，它是怎么work起来的呢？'


+++


![](cover.jpg)


## 1. 为什么我想聊聊这个？
在从Java转向Go后，有一个我比较迷惑的点，为什么我在编译时，会自动生成wire_gen文件和gen_provider文件，多了一道工序，它有什么用呢？

因为，这与我之前Java使用的Spring容器似乎不太一样，Spring容器是动过类加载的方式，加载并通过反射来实例化出各个bean的，所有的依赖都是自动注入，非常方便。而Go的Wire框架，看起来要先通过像是静态编译的方式，通过go generate来收集组件都依赖了哪些其他组件，然后再对各组件进行实例化。

当然，这些都是我的推测，Wire框架它对我是一个黑盒。这导致我在使用Wire框架时不够熟练，比如我要删除一个组件，那么我应该更改哪些配置文件，应该重新执行哪些指令，来自动生成哪些文件。那么这篇blog，我就是想从底层原理出发，对Wire框架产生一个详细的认知。毕竟“工欲善其事，必先利其器”嘛！

那么，让我们开始吧。


## 2. Wire框架是否能类比成Spring框架？
在一开始，我想确认Wire框架究竟是做什么用的。我比较熟悉Spring框架，同时按照我当下浅薄的认知，我觉得他们的定位相似。那么我们就类比两者，来一步步了解Wire框架。

那么，我想从「配置方式」、「注入时机」、「依赖检查」、「运行时容器」四个方面分别了解。

### 2.1 配置方式
对于Java，通过@Bean、@Component系列注解，来声明Spring Bean；同时，在使用到这些Bean的地方，通过@Autowire、@Resource等注解就可以完成注入。
而对于Go，则需要指定以下配置：
- Provider：它是定义 “如何创建对象” 的函数，每个 Provider 函数负责实例化一个具体对象。若该对象依赖其他对象，直接在函数参数中声明依赖。
- Injector：定义需要组装哪些对象。形象一点说，Provider告诉你做一道菜需要哪些食材，而Injector则是你真正要点的菜的清单。你会按需点菜。
- wire_gen：它是Wire框架自动生成的，它能帮助我们自动生成「调用Provider」的实现。用上面的例子，相当于根据你的订单还有菜品做法，自动生成一套做饭指令，等待执行。
至此，Wire框架配置已完成。


### 2.2 注入时机
对于Java，在服务启动时，会扫描并加载Spring Bean。当该Bean实例化完成后，会被注入到Spring容器中。如果在实例化的时候，涉及到其他依赖的Bean，那么会从容器中取出该成员Bean（或者执行成员Bean的实例化），并根据对成员Bean的依赖，将其注入到当前Bean中。整体是在服务启动时完成。

而对于Wire框架，则并不是这样的，根本原因是Wire它并不是一个容器，它本质上是一个编译时依赖注入工具。
- 为什么这么说，因为Wire的注入逻辑，在服务启动之前就已经确定，我们通过代码生成的方式生成wire_gen；然后，在编译阶段，相关逻辑就会嵌入可执行文件中了。
- 在运行阶段，直接调用可执行文件中的注入器函数，获取已组装好的对象。
而这也是Go启动为什么那么快的原因。没有反射，纯函数调用！


### 2.3 依赖检查
因为注入时机的不同，Spring框架和Wire框架的依赖检查的时机也完全不同。

对于Java，它会在解析@Autowired等注入注解时，去尝试分析Bean之间的依赖关系，检查所依赖的Bean是否能初始化或者已经注入到了Spring容器中。如果存在依赖缺失，会出现 `NoSuchBeanDefinitionException` 类似的异常信息；而如果出现了循环依赖，且无法解决，则会出现 `BeanCurrentlyInCreationException` 类似的异常信息。相关的依赖问题，不会在编译器出现。

而对于Wire框架，依赖的检查是在编译前，也就是wire_gen在生成时，会进行校验。在生成wire_gen的时候，会静态分析所有Provider的依赖关系，尝试构建依赖链。如果存在依赖缺失，则Wire直接报错并返回缺失提醒；如果出现了循环依赖，会直接报错并提示循环依赖。


### 2.4 运行时容器
分析完上面三点后，我有个大大的问题，Wire框架中某组件被多次注入到不同组件的时候，每次都是通过Provider创建出一个新对象么？

因为对于Java，它持有了一个运行时容器BeanFactory，所有的Bean都往里面注入，随用随取，BeanFactory管理了所有Spring Bean的生命周期。整体采用单例的设计模式。

然后我调研了一下Wire框架，它没有任何容器的概念，或者说，Wire框架只是作用于编译阶段，到运行阶段后它就“消失”了。
那么，回到开始的问题，在编译阶段，每次注入某组件时，都会调用Provider方法吗？是的，如果不做特殊处理，那么Wire注入的组件是多例。
官方对此也有说明：
> (Reference [guide.md](https://github.com/google/wire/blob/main/docs/guide.md))  
> Providers are just functions. They can be simple constructors, or they can have side effects, or they can return singletons. Wire doesn't care; it just calls the functions as needed. 


## 3 Wire框架的设计理念的优劣势
我觉得整理一下Wire框架的优劣势，这对我未来做技术选型是有帮助的。

我认为Wire框架在以下方面是有优势的：
1. 最重要的是依赖关系清晰，Spring容器相对黑盒，提供便捷性的同时，逻辑比较复杂；而Wire比较简单粗暴，依赖出现问题直接断点调试。
2. 性能好，wire是通过代码生成实现的依赖注入，没有Spring容器的反射、动态代理、Bean池管理等运行时开销。
3. 编译时就进行依赖检查，能提前暴露问题。

但我认为也有劣势：
1. 多了wire_gen生成这一步，每次增加组件后，都需要执行这一步，有一丢丢繁琐。
2. 需要手动管理单例，如果能像Spring容器，通过一个@Scope自动管理就好了。
3. 灵活性比较差，因为依赖关系都是固定死的，有时需要根据环境进行动态注册，Wire是支持不了的，而Spring可以通过@Conditional来实现。

到这为止，我可以给Wire框架下一个定义了：

**Wire就是一个DI框架，只负责如何组装依赖以及做依赖注入代码的生成，像Spring的IOC、AOP这些能力是不支持的。**


## 4. provider_gen.go是做啥的？
我在团队项目下，发现provider_gen.go文件，看名字并不是手写。它是怎么生成的？为了解决什么问题？

首先，它并不是wire框架生成的，而是利用go generate工具链生成的。这实际上是Go一个最佳实践，针对项目中大量重复且功能一致的对象，可以通过模版的方式批量创建。比如，针对DAO层的对象，其模版可以是——
```
// 模板：dao_provider.tpl
package dao

import "your/project/db"

// New{{.Name}}DAO 初始化 {{.Name}}DAO
func New{{.Name}}DAO(db *db.DB) *{{.Name}}DAO {
  return &{{.Name}}DAO{
    db: db,  // 固定依赖数据库连接
  }
}
```

其实就是一个简化操作的作用。当我们创建好了模版，并指定了使用该模版的对象，使用go generate就能一键生成Provider方法，这些方法的实现会被放置到provider_gen.go中。


## 5. di.go是做啥的？
同样的，在团队项目下，也发现了di.go文件，它是怎么生成的呢？
我们在Go项目中往往会定义一个InitializeXXX的注入函数，并使用 wire.Build 函数指定需要组合的 Provider。当我们执行Wire工具，就会自动生成di.go，具体流程是这样的：
- 解析 wire.Build 中指定的所有 Provider 方法
- 构建依赖关系图（分析每个 Provider 的参数依赖）
- 生成一个完整的初始化函数，替代手动定义的 InitializeApp
- 将生成的代码写入 di.go 文件
这里，InitializeXXX往往会变大很多，因为它会增加底层的依赖的provider调用。


