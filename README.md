# The end of the API framework

English | [简体中文](README.zh.md)

## Contents

- [Improvements to the Grape framework](#improvements-to-the-grape-framework)
  * [Deep `expose`](#deep-expose)
  * [The response entity](#the-response-entity)
  * [Integration of parameters and entities](#integration-of-parameters-and-entities)
  * [Improvements for testing](#improvements-for-testing)
- [How to get started](#how-to-get-started)
- [The ideas](#the-ideas)
  * [Document-oriented development](#document-oriented-development)
  * [Do you really no need Controller testing?](#do-you-really-no-need-controller-testing)
- [The end](#the-end)

## Improvements to the Grape framework

### Deep `expose`

As you may know, `expose` can be nested. If a block without parameters is passed, nested rendering will be performed:

```ruby
class Entities::Article <Grape::Entity
  expose :user do
    expose :name {|article| article.user.name}
    expose :age {|article| article.user.age}
  end
end
```

As above, a data structure like the following will be rendered:

```json
{
  "user": {
    "name": "Jim",
    "age": 18
  }
}
```

If `deep: true` is passed in at this time, the instance bound to the nesting layer will be different. The following example shows the same effect as the above one:

```ruby
class Entities::Article <Grape::Entity
  expose :user, deep: true do
    expose :name
    expose :age
  end
end
```

### The response entity

`status` is used to declare the response entity (`success`, `fail`, `entity` are actually aliases with special circumstances). It has the following invoking forms:

```ruby
status 200 do
  expose :article, using: Entities::Article
end

status 200,'Return article data' do
  expose :article, using: Entities::Article
end

status 400,'The request parameter is wrong' do
  expose :code, desc: 'error code'
  expose :message, desc: 'error message'
end

success'Request is successful' do
  expose :article, using: Entities::Article
end

fail'Request failed' do
  expose :code, desc: 'error code'
  expose :message, desc: 'error message'
end

entity do
  expose :article, using: Entities::Article
end
```

The above code mainly serves two aspects:

1. Call the `present` method in the interface logic without explicitly specifying the `Entity` type, which is automatically resolved.

   Previously, you had to write as following:

   ```ruby
   present :article, article, with: Entities::Article
   ```

   For now, just write it as:

   ```ruby
   present :article, article
   ```

   Because the `status` declaration has already known how to render the `article` entity.

2. It can generate corresponding documentation based on `status` DSL.

### Integration of parameters and entities

It adds a new method `to_params` in `Grape::Entity`, allowing you to reuse it in parameter declarations:

```ruby
params do
  requires :article, type: Hash do
    optional :all, using: Entities::Article.to_params
  end
end
```

It is better than `Grape::Entity.documentation`, with the following improvements:

1. `type` can be written as string:

   ```ruby
   expose :foo, documentation: { type:'string' }
   ```

2. Some additional parameters can be mixed into `documentation` hash option, such as `param_type`:

   ```ruby
   expose :foo, documentation: { param_type:'body' }
   ```

3. Properly process `is_array` option:

   ```ruby
   expose :bar, documentation: { type: String, is_array: true }
   ```

### Improvements for testing

Now you can use the programming style to test the return value of the interface, no need to test such as JSON string, XML text and so on. If you implement the interface like this:

```ruby
present :article, article
```

You can test it just like:

```ruby
get'/article/1'
assert_equal 200, last_response.status
assert_equal articles(:one), presents(:article)
```

Pay attention to the `presents` method, it is the magical artifact I provide for you.

**Note that this function is not implemented in the framework, you need to clone the scaffolding project:**

```bash
git clone https://github.com/run27017/grape-app-demo.git
```

## How to get started

If you are a complete novice, it is recommended to get familiar with the [Grape](https://github.com/run27017/grape.git) framework first. I suggest you read the [document](https://github.com/run27017/grape/blob/master/README.md) of my forked repository. After you are very familiar with the Grape framework, read the above part of my improvements to the Grape framework. For the design concept of the entire framework, you can read the following content of this article.

You can start a project started with my scaffolding project, and all features are already integrated:

```bash
git clone https://github.com/run27017/grape-app-demo.git
```

## The ideas

### Document-oriented development

Under current era, there are many development paradigms for back-end developers to choose from, such as *test-driven development*, *behavior-driven development*, *agile software development* and so on. In contrast, I proposed a new idea, which is called *document-oriented development*.

When writing an API project, it is necessary to prepare documentation. I don't know how everyone prepares the documentation, but usually there is a strange circle: I have to repeat myself both in writing logic code and writing documentation. why can't I have completed the document synchronously while writing the interface logic? If I can invent a DSL that can control the behavior of the interface while writing a document, isn't that what I want?

Just do it!

I found that [Grape](https://github.com/ruby-grape/grape) framework already provides a similar DSL. For example, you can specify parameters like this:

```ruby
params do
  requires :user, type: Hash do
    requires :name, type: String, desc:'name of user'
    requires :age, type: Integer, desc:'age of user'
  end
end
```

The above code can restrict the parameters to the two fields of `name` and `age`, and restricting their types to `String` and `Integer` respectively. At the same time, one library called [grape-swagger](https://github.com/ruby-grape/grape-swagger.git) can render the macro definition of `params` as part of the Swagger document. Documentation and implementation are combined perfectly.

In addition, the Grape framework provides the `desc` macro, which is a pure document statement for third-party libraries to read and will not affect the interface behavior in any way.

```ruby
desc 'Create new user' do
  tags 'users'
  entity Entities::User
end
```

However, the Grape framework is not a complete document-oriented development framework. It has many important missions, so the seamless connection between it and the document is limited to these. As you can see, the `params` macro is a perfect example, the `desc` macro is unfortunately only related to document rendering, and then nothing else.

Since the Grape framework is an open source framework, it is easy to modify it to add a few new functions. It took me a few days to add a `status` macro, which can be used to declare the response entity:

```ruby
status 200 do
  expose :user, deep: true do
    expose :id, documentation: { type: Integer, desc:'id of user' }
    expose :name, documentation: { type: String, desc:'name of user' }
    expose :age, documentation: { type: Integer, desc: 'age of user' }
  end
end
```

The above code mainly serves two aspects:

1. Call the `present` method in the interface logic without explicitly specifying the `Entity` type, which is automatically resolved.

   Previously, you had to write as following:

   ```ruby
   present :article, article, with: Entities::Article
   ```

   For now, just write it as:

   ```ruby
   present :article, article
   ```

   Because the `status` declaration has already known how to render the `article` entity.

2. It can generate corresponding documentation based on `status` DSL.

Everything is just the tip of the iceberg.

### Do you really no need Controller testing?

Regarding the unit testing of interfaces, there are two points to debate: Which is more reasonable, the integration testing or the controller testing? Integration testing is like a black box. Developers call the interface and then test the view returned by the interface. The Controller test will also call the interface in the same way, but will test the internal state.

Through the following two cases, you can intuitively feel the differences of controller test and integration test in Rails.

In earlier versions of Rails, it exists controller test:

```ruby
class ArticlesControllerTest <ActionController::TestCase
  test "should get index" do
    get :index
    assert_response :success
    assert_equal users(:one, :two, :three), assigns(:articles)
  end
end
```

After Rails 5, it recommender integration testing more:

```ruby
class ArticlesControllerTest <ActionDispatch::IntegrationTest
  test "should get index" do
    get articles_url
    assert_response :success
  end
end
```

Note that there is no corresponding `assert_equal` statement in the Integration test, it is because writing  it is very difficult. For example, if the view returns JSON data, you can try the following equivalent code:

```ruby
assert_equal users(:one, :two, :three).as_json, JSON.parse(last_response.body)
```

But it is often fail because it closely depend on view changes. The above one is just a try.

I don't want to purpose a discussion  of which is better between controller test and integration test, even you may have been aware of my tendency from all I have written. This topic can be discussed from 2012 to 2020. If you don’t believe me, you can read  [this post](https://github.com/betterspecs/betterspecs/issues/14). I don't want to make it worse. 

Maybe those refusing controller testing consider it is not elegant, because it need to invest the instance variables, which is the internal state of controller test. Fortunately, I did some simple work to make it more elegant to test it. You only need to specify the rendering data using the `present` method in the logical interface:

```ruby
present :users, users
```

Then test it with the special `presents` method in the test case:

```ruby
assert_equal users(:one, :two, :three), presents(:users)
```

It's similar to `assigns`, but it's more comfortable, isn't it?

## The end

This is my transformation of the Grape framework, which has already begun and will continue. My transformation concept is nothing more than two ideas: better documentation integration and better testing. In fact, it only takes a little time to make it work.

If you also want to use the inproveoment version of the Grape framework, just clone my scaffold directly:

```bash
git clone https://github.com/run27017/grape-app-demo.git
```

The scaffold uses the frame set of my fork. They are:

- [grape](https://github.com/run27017/grape)
- [grape-entity](https://github.com/run27017/grape)
- [grape-swagger](https://github.com/run27017/grape-swagger)

Click the links and have a look, maybe you can also become a participants in open source world.