# Object creation with activesupport `delegate`

Today I was working on adding the possibility to refresh an access_token tou our rails app.
We use doorkeeper and they already have a [configuration option](https://github.com/doorkeeper-gem/doorkeeper/blob/v5.2.3/lib/generators/doorkeeper/templates/initializer.rb#L152) for that. It worked locally perfect when testing against my 0Auth2 client.

Yet when trying to create some requests spec's and using my access_token factory the `refresh_token` field was always `nil`

I knew from digging into the [`access_token` generation](https://github.com/doorkeeper-gem/doorkeeper/blob/v5.2.3/lib/doorkeeper/orm/active_record/access_token.rb#L19) that was done via rails hook. I noticed that the was also the case for the `refresh_token`. 
Yet there was more addition to it. For the `refresh_token` there was additionally an `if` condition which determent whether my `refresh_token` will be generated or not.

```ruby
before_validation :generate_refresh_token, on: :create, if: :use_refresh_token?
```


After a while I just could not figure out why this condition was always failing in my spec. So I ask a colleague for help.

We discovered that value `@use_refresh_token` for the condition was coming from the outside world.
```ruby
def use_refresh_token?
  @use_refresh_token ||= false
  !!@use_refresh_token
end
```

When also passing the value from our factory the token `refresh_token` was created as expected.



In order to understand the root cause for the behavior I took a deeper into the doorkeeper gem.

I found out that the an access_token is generated the following way:
1. [Hit the controller](https://github.com/doorkeeper-gem/doorkeeper/blob/v5.2.3/app/controllers/doorkeeper/tokens_controller.rb#L7) from the engine
2. Inside the controller [try to authorize the request](https://github.com/doorkeeper-gem/doorkeeper/blob/v5.2.3/app/controllers/doorkeeper/tokens_controller.rb#L93)
3. In order to authorize the request [find out which strategy `grant_type`](https://github.com/doorkeeper-gem/doorkeeper/blob/v5.2.3/app/controllers/doorkeeper/tokens_controller.rb#L87)  is used
4. Now some meta programming to [create the correct strategy class](https://github.com/doorkeeper-gem/doorkeeper/blob/v5.2.3/lib/doorkeeper/server.rb#L16) on the fly
5. [call `authorize`](https://github.com/doorkeeper-gem/doorkeeper/blob/v5.2.3/app/controllers/doorkeeper/tokens_controller.rb#L93) to check if the token can be issued
6. This `authorize` method [gets delegated](https://github.com/doorkeeper-gem/doorkeeper/blob/v5.2.3/lib/doorkeeper/request/strategy.rb#L8)
7. [The target method of the delegation](https://github.com/doorkeeper-gem/doorkeeper/blob/v5.2.3/lib/doorkeeper/request/authorization_code.rb#L8) than creates create's a class
8. This class is a [child class](https://github.com/doorkeeper-gem/doorkeeper/blob/v5.2.3/lib/doorkeeper/oauth/authorization_code_request.rb#L5) in which the [parent class has original method defined](https://github.com/doorkeeper-gem/doorkeeper/blob/v5.2.3/lib/doorkeeper/oauth/base_request.rb#L10)

Now some more things happen in which I didn't look into anymore
As this example is very complex I created a simple example which does the same thing.

```ruby
require 'active_support/core_ext/module/delegation'

class LazyEmployee
  delegate :make_me_a_sandwich, to: :sandwich_maker

  def sandwich_maker
    @sandwich_maker ||= SandwichMaker.new()
  end
end

class SandwichMaker
  def make_me_a_sandwich
    puts "this is a cool method"
  end
end
```

To use it just open a rails console `rails c`
```ruby
[1] pry(main)> lazy_employee  = LazyEmployee.new
=> #<LazyEmployee:0x00007f8a4b3e6188>
[2] pry(main)> lazy_employee.make_me_a_sandwich
this is a cool method
=> nil
```


### Credits
* [Kamil Lelonek](https://twitter.com/kamillelonek) for [explaining delegation](https://blog.lelonek.me/how-to-delegate-methods-in-ruby-a7a71b077d99)
* [Reid Beels](https://twitter.com/reidab) to for adding [this trick](https://github.com/doorkeeper-gem/doorkeeper/commit/6eb9b2d6e9dd3e94e6d8c4e9362a3e085b7e15a4#diff-439bd91f0a870bd3ac57594b186a40c1) to the doorkeeper gem
