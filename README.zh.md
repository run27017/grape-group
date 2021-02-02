# API 框架的终结

[English](README.md) | 简体中文

内容目录
-----------------

- [针对 Grape 框架的改进](#%E9%92%88%E5%AF%B9-grape-%E6%A1%86%E6%9E%B6%E7%9A%84%E6%94%B9%E8%BF%9B)
  * [深层 `expose`](#%E6%B7%B1%E5%B1%82-expose)
  * [接口返回值声明](#%E6%8E%A5%E5%8F%A3%E8%BF%94%E5%9B%9E%E5%80%BC%E5%A3%B0%E6%98%8E)
  * [参数、返回值一体化](#%E5%8F%82%E6%95%B0%E8%BF%94%E5%9B%9E%E5%80%BC%E4%B8%80%E4%BD%93%E5%8C%96)
  * [针对测试的改进](#%E9%92%88%E5%AF%B9%E6%B5%8B%E8%AF%95%E7%9A%84%E6%94%B9%E8%BF%9B)
- [新手如何上手](#%E6%96%B0%E6%89%8B%E5%A6%82%E4%BD%95%E4%B8%8A%E6%89%8B)
- [理念](#%E7%90%86%E5%BF%B5)
  * [面向文档的开发](#%E9%9D%A2%E5%90%91%E6%96%87%E6%A1%A3%E7%9A%84%E5%BC%80%E5%8F%91)
  * [你真的不需要 Controller 测试吗？](#%E4%BD%A0%E7%9C%9F%E7%9A%84%E4%B8%8D%E9%9C%80%E8%A6%81-controller-%E6%B5%8B%E8%AF%95%E5%90%97)
- [写在最后](#%E5%86%99%E5%9C%A8%E6%9C%80%E5%90%8E)

## 针对 Grape 框架的改进

### 深层 `expose`

你也许知道，`expose` 可以嵌套。如果传递的是个不带参的块，则会执行嵌套渲染：

```ruby
class Entities::Article < Grape::Entity
  expose :user do
    expose :name { |article| article.user.name }
    expose :age { |article| article.user.age }
  end
end
```

如上，会渲染一个如下的数据结构：

```json
{
  "user": {
    "name": "Jim",
    "age": 18
  }
}
```

如果此时传入 `deep: true`，则嵌套层绑定的 instance 就不一样了。下面的例子与上面的例子展示了同样的效果：

```ruby
class Entities::Article < Grape::Entity
  expose :user, deep: true do
    expose :name
    expose :age
  end
end
```

### 接口返回值声明

`status` 用于声明接口的返回值（`success`、`fail`、`entity` 是特殊情况下的别名）。它有如下几种基本的调用形式：

```ruby
status 200 do
  expose :article, using: Entities::Article
end

status 200, '返回文章数据' do
  expose :article, using: Entities::Article
end

status 400, '请求参数有误' do
  expose :code, desc: '错误码'
  expose :message, desc: '错误消息'
end

success '请求成功' do
  expose :article, using: Entities::Article
end

fail '请求失败' do
  expose :code, desc: '错误码'
  expose :message, desc: '错误消息'
end

entity do
  expose :article, using: Entities::Article
end
```

上述声明主要起两个作用：

1. 在接口逻辑中调用 `present` 方法不用显示地指定 `Entity` 类型，它是自动解析的。

   以前，你必须这样调用：

   ```ruby
   present :article, article, with: Entities::Article
   ```

   现在，只用这样：

   ```ruby
   present :article, article
   ```

   因为在 `status` 声明中它已经知道如何渲染 `article` 实体了。

2. `status` 声明可对应生成文档。

### 参数、返回值一体化

`Grape::Entity` 新加一个 `to_params` 方法，使得你可以在参数声明中重复利用：

```ruby
params do
  requires :article, type: Hash do
    optional :all, using: Entities::Article.to_params
  end
end
```

它比 `Grape::Entity.documentation` 更好用，做了如下改进：

1. `type` 可以写成字符串：

   ```ruby
   expose :foo, documentation: { type: 'string' }
   ```

2. 可混用额外参数，如同时指定 `param_type` 参数：

   ```ruby
   expose :foo, documentation: { param_type: 'body' }
   ```

3. 合理处理了 `is_array`：

   ```ruby
   expose :bar, documentation: { type: String, is_array: true }
   ```

### 针对测试的改进

现在可以用编程 style 的方式测试接口的返回值了，不需要再测试诸如 JSON 字符串、XML 文本之类的。如果像下面这样实现接口：

```ruby
present :article, article
```

就可以像下面这样测试：

```ruby
get '/article/1'
assert_equal 200, last_response.status
assert_equal articles(:one), presents(:article)
```

注意到以上的 `presents` 方法，这是我为你特别定制的测试神器。

**这个功能不是实现在框架中的，需要克隆脚手架项目：**

```bash
git clone https://github.com/run27017/grape-app-demo.git
```

## 新手如何上手

如果你是完全的新手，建议先从熟悉 [Grape](https://github.com/run27017/grape.git) 框架开始。我建议您阅读我仓库下的[文档](https://github.com/run27017/grape/blob/master/README.md)。在您熟悉了 Grape 框架以后，再阅读以上我对 Grape 框架改进的部分。关于整个框架的设计理念，可阅读后文。

项目上手就从我提供的脚手架项目开始，所有功能都集成进来了：

```bash
git clone https://github.com/run27017/grape-app-demo.git
```

## 理念

### 面向文档的开发

当今环境下，有许多的开发范式供后端开发者选择，例如*测试驱动开发*、*行为驱动开发*、*敏捷软件开发*等等。与之相对的，我提出了一个新的想法，我将其称为*面向文档的开发*。

写 API 项目的同时是要准备文档的。我不知道大家是如何准备文档的，但往往都逃不出一个怪圈：同一个接口，我的实现要写一份，我的文档也要同时写一份。我常常在想，为什么我在写接口的同时不能将文档同步地完成呢？换个角度想，接口文档的契约精神为何不能自动地成为实现的一部分？如果，我能发明一个 DSL，在编写文档的同时就能够制约接口的行为，那不正是我想要的吗？

说干就干！

我发现 [Grape](https://github.com/ruby-grape/grape) 框架就已经提供了类似的 DSL 了。例如你在制定参数时可以像这样：

```ruby
params do
  requires :user, type: Hash do
    requires :name, type: String, desc: '用户的姓名'
    requires :age, type: Integer, desc: '用户的年龄'
  end
end
```

上面的代码就可以对参数进行制约，限制参数为 `name` 和 `age` 两个字段，并分别限制它们的类型为 `String` 和 `Integer`. 与此同时，一个名为 [grape-swagger](https://github.com/ruby-grape/grape-swagger.git) 的库可以将 `params` 的宏定义渲染成 Swagger 文档的一部分。完美，文档和实现在这里结合了。

另外，Grape 框架提供了 `desc` 宏，它是一个纯文档的声明供第三方库读取，不会对接口行为产生任何影响。

```ruby
desc '创建新用户' do
  tags 'users'
  entity Entities::User
end
```

但是，毕竟 Grape 框架不是完全的面向文档的开发框架，它有很多重要的使命，所以它和文档的无缝衔接也就仅限于此了。你能看到，`params` 宏是个完美结合的范例，`desc` 宏很可惜只与文档渲染有关，然后就别无其他了。

鉴于 Grape 框架是个开源框架，修改它以添加几个小零件还是很简易的一件事。我用了几天的时间添加了一个 `status` 宏，可以用它来声明返回值：

```ruby
status 200 do
  expose :user, deep: true do
    expose :id, documentation: { type: Integer, desc: '用户的 id' }
    expose :name, documentation: { type: String, desc: '用户的姓名' }
    expose :age, documentation: { type: Integer, desc: '用户的年龄' }
  end
end
```

上述声明主要起两个作用：

1. 在接口逻辑中调用 `present` 方法不用显示地指定 `Entity` 类型，它是自动解析的。

   以前，你必须这样调用：

   ```ruby
   present :user, user, with: Entities::User
   ```

   现在，只用这样：

   ```ruby
   present :user, user
   ```

   因为在 `status` 声明中它已经知道如何渲染 `user` 实体了。

2. `grape-swagger` 库可以解析 `status` 宏生成文档。

一切还只是冰山一角。

### 你真的不需要 Controller 测试吗？

有关接口的单元测试，有两个观点争论不休：接口测试应该是针对 Integration 测试还是 Controller 测试？Integration 测试像是一个黑匣子，开发者调用接口，然后针对接口返回的视图进行测试。而 Controller 测试也会一样地调用接口，但会测到内部的状态。

通过下面的两个案例直观地感受一下 Controller 测试和 Integration 测试在 Rails 框架中的不同写法。

在早期的 Rails 版本中，是有 Controller 测试的：

```ruby
class ArticlesControllerTest < ActionController::TestCase
  test "should get index" do
    get :index
    assert_response :success
    assert_equal users(:one, :two, :three), assigns(:articles)
  end
end
```

Rails 5 以后，更推荐 Integration 测试：

```ruby
class ArticlesControllerTest < ActionDispatch::IntegrationTest
  test "should get index" do
    get articles_url
    assert_response :success
  end
end
```

注意到 Integration 测试中是没有对应的 `assert_equal` 语句的，因为它比较难写。如果视图返回的是 JSON 数据，可以尝试用下面的等效写法：

```ruby
assert_equal users(:one, :two, :three).as_json, JSON.parse(last_response.body)
```

但是测试视图的代码及其依赖视图的变化，也会经常性的失效，上面只是尝试性的一种写法而已。

我不想在这里探讨 Controller 测试和 Integration 测试究竟孰优孰劣的问题，尽管你可能已经从字里行间察觉到我的倾向性。关于这个话题能够从 2012 年讨论到 2020 年，不信你可以看这个[帖子](https://github.com/betterspecs/betterspecs/issues/14)。我可不想让情况变得更糟。很多次我在想，那些反对 Controller 测试的人可能仅仅把 Controller 的实例变量看成是其内部状态而已，而没有意识到它也是 Controller 与 Template 之间的契约。这不怪他们，因为用传递实例变量的方式定义契约确实不够优雅。

好在我做了一些简单的工作，让其在 Grape 框架内可以更优雅地测试接口的返回。你只需要在逻辑接口中使用 `present` 方法指定渲染数据：

```ruby
present :users, users
```

然后就可以在测试用例中结合特殊的 `presents` 方法测试它：

```ruby
assert_equal users(:one, :two, :three), presents(:users)
```

跟 `assigns` 很像，但是它更舒服不是么？

## 写在最后

这就是我对 Grape 框架的改造过程，已经开始并将持续下去。我的改造理念无外乎两个想法：更好的文档结合和更好的测试。而事实上，只需要一点点工作，确实就可以起作用了。

如果你也想用到我所改造的 Grape 框架，直接克隆我的脚手架就好了：

```bash
git clone https://github.com/run27017/grape-app-demo.git
```

脚手架中使用的是我 Fork 后的框架集，它们是：

- [grape](https://github.com/run27017/grape)
- [grape-entity](https://github.com/run27017/grape)
- [grape-swagger](https://github.com/run27017/grape-swagger)

点击它们的链接看一看吧，也许你也能成为开源世界的参与者呢。