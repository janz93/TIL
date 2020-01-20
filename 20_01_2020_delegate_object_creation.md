# Object creation with activesupport delegate

Today I was working on adding the possibility to refresh a access_token in our rails app.
We use doorkeeper and they already have a configuration option for that. It worked perfect when testing against my 0Auth2 client.

Yet when trying to create some requests spec's and using my access_token factory the refresh_token field was always `nil`

I knew from playing around with the token value that the token is [generated via rails hook]() and I noticed that the was also the case for the refresh_token. 
Yet there was more addition to it. On the refresh_token there was a `if` condition which determent whether my refresh_token will be generated or not.

After hitting a deadend by myself I ask a colleague for help.

We discovered that value for the `if` condition was coming from the outside world and when passing the value also trough our factory the token was created as expected.

In order to understand the root cause for the behavior I dug a bit deeper into the doorkeeper gem.

I found out that the access_token is generated the following way
1. [Hit the controller from the engine](https://github.com/doorkeeper-gem/doorkeeper/blob/v5.2.3/app/controllers/doorkeeper/tokens_controller.rb#L7)
2. [Inside the controller try to authorize the request](https://github.com/doorkeeper-gem/doorkeeper/blob/v5.2.3/app/controllers/doorkeeper/tokens_controller.rb#L93)
3. [In order to authorize the request find out which strategy `grant_type` was used](https://github.com/doorkeeper-gem/doorkeeper/blob/v5.2.3/app/controllers/doorkeeper/tokens_controller.rb#L87)
4. [Meta programming to create the correct strategy class on the fly](https://github.com/doorkeeper-gem/doorkeeper/blob/23e9c0316a24c28819f8b194a113fb7bf5b935ba/lib/doorkeeper/server.rb#L16)
5. [call `authorize` to check if with the the token can be issued](https://github.com/doorkeeper-gem/doorkeeper/blob/v5.2.3/app/controllers/doorkeeper/tokens_controller.rb#L93)
6. [This method get delegated](https://github.com/doorkeeper-gem/doorkeeper/blob/ad7e17bbf0696e2516a433e06b346d9468386f47/lib/doorkeeper/request/strategy.rb#L8)
7. [The target method of the delegation than creates create's a class](https://github.com/doorkeeper-gem/doorkeeper/blob/ad7e17bbf0696e2516a433e06b346d9468386f47/lib/doorkeeper/request/authorization_code.rb#L8)
8. [This class is a child class in which the parent has original method defined](https://github.com/doorkeeper-gem/doorkeeper/blob/ad7e17bbf0696e2516a433e06b346d9468386f47/lib/doorkeeper/oauth/base_request.rb#L10)

Now some more thing happen in which I didn't look into anymore
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
