---
layout: page
title: Outputting Text to Logger for Debugging
section: Debugging
---

The first and most widely used method of debugging Ruby applications is simply outputting text data. Let's look at a few approaches at increasing levels of expertise.

## Reading a Request/Response

Greatly undervalued by newer Rails developers, the first step when observing unexpected behavior should be the server log and the request/response information. Let's pick apart an example.

### Sample Request/Response

Here's an actual request from a sample application:

{% terminal %}
Started POST "/products" for 127.0.0.1 at 2011-07-19 12:42:48 -0700
  Processing by ProductsController#create as HTML
  Parameters: {"utf8"=>"✓", "authenticity_token"=>"E3aGszU+Xmdq3bs9woAtMzH93zy34Z9lQqiNHwLACRY=", "product"=>{"title"=>"Apples", "price"=>"5.99", "stock"=>"12", "description"=>"A bag of apples.", "image_url"=>"apples.jpg"}, "commit"=>"Create Product"}
  User Load (0.3ms)  SELECT "users".* FROM "users" WHERE "users"."id" IS NULL LIMIT 1
  Order Load (1.1ms)  SELECT "orders".* FROM "orders" WHERE "orders"."id" = 48 LIMIT 1
  AREL (0.5ms)  INSERT INTO "products" ("title", "price", "description", "image_url", "created_at", "updated_at", "stock") VALUES ('Apples', 5.99, 'A bag of apples.', 'apples.jpg', '2011-07-19 19:42:48.457003', '2011-07-19 19:42:48.457003', 12)
Redirected to http://localhost:3000/products/3
Completed 302 Found in 117ms
{% endterminal %}

Here are some of the facts we can learn from reading the request:

* Line1: It executed a `POST` request for `/products` which should trigger the `create` action of `ProductsController`. Is this the controller and action intended? Especially scrutinize the request verb, this is a common place for mixups.
* Line2: It did attempt to execute `ProductsController#create`
* Line3: The params hash. Look for elements that are blank/nil. Is the structure reasonable? Unfortunately, because of the UTF-8 checkmark, this hash can't be copy/pasted into the console for testing unless you're on Ruby 1.9.2. If you have trouble navigating this hash:
  * Copy it from the log output
  * Paste it into a text editor
  * Remove the `utf8` key/value pair
  * Copy the result
  * Paste it into console and experiment
  * Inspect whether attributes are correctly nested, correctly named, etc
* Lines 4-6: See the SQL interactions. Particularly pay attention to `INSERT` statements. Is the information from the Line 3 params in the `INSERT` or are there a bunch of `nil` fields? The most common issue you'll spot here is this:
  ```text
    WARNING: Can't mass-assign protected attributes: [your attributes here]
  ```
  In that case, look in the model for the `attr_accessible` method call and see if additional fields should be made accessible. Note that the suffix matters, so making `product` accessible will not automatically accept `product_id`, list both to allow mass-assignment via ID number or actual object.
* Last Lines: Redirect or render and HTML status code. Usually not helpful unless the app is redirecting you somewhere unexpected.

Just to say it one more time, the issue about *Can't mass-assign protected attributes* is an incredibly common issue new Rails developers spend hours debugging. If you see the error in the log, though, it can be fixed in seconds. Just go to your model file and add the attributes to `attr_accessible`.

## Temporary Instructions

Let's next look at adding temporary instructions to our application.

### Using Warn

Looking at the server log gave lots of great info, but it doesn't help with inspecting the values of variables or instructions. For that job, use the `warn` method of Ruby's `Kernel` object. Here's how you might insert it into the `create` action used above:

```ruby
def create
  @product = Product.new(params[:product])
  warn "Product before save:"
  warn @product.inspect
  if @product.save
    redirect_to @product, notice: "Successfully created product."
  else
    render action: 'new'
  end
end
```

Then in the output log see the results:

{% terminal %}
Product before save:
#<Product id: nil, title: "Apples", price: nil, description: nil, image_url: nil, created_at: nil, updated_at: nil, stock: 0>

  Started POST "/products" for 127.0.0.1 at 2011-07-19 13:18:26 -0700
    Processing by ProductsController#create as HTML
{% endterminal %}

Notice that the warn comes out before where the log claims it is "starting" the response. The `warn` is output immediately, while the normal logging operations are buffered and output all together.

<div class="opinion">
<p>When I use warn I'll typically put in some label to the output, like the <code>Product before save</code> here. The messages for <code>warn</code> are just strings, so you can use <code>\n</code> newlines or other text formatting to make them easier to read.
</div>

### Raising Exceptions

One of my most frequently used techniques is to intentionally raise an exception. If I wanted to check out the `@product` object during the `create` action and maybe look at the parameters of the request, I'd typically do this:

```ruby
def create
  @product = Product.new(params[:product])
  raise @product.inspect
  if @product.save
  #...
```

The `raise` will immediately halt execution and display Rails' normal error page. The `raise` method accepts one parameter, a string, which will be output as the error message.

With this usage you'd see something like this:

{% terminal %}
RuntimeError in ProductsController#create
#<Product id: nil, title: "Apples", price: nil, description: nil, image_url: nil, created_at: nil, updated_at: nil, stock: 0>
{% endterminal %}

The first line specifying that it was a general `RuntimeError` exception and the second line is the message, the result of our `inspect`. Generally `inspect` is a better choice than `to_s` as it'll show more about the object's internal state.

Also, further down the page you'll see the request's parameters formatted in a YAML debug block.

This is a great debugging technique when writing Rails applications because you don't have to dig through anything -- execution halts right at your message.

You can even use `raise` in conjunction with Ruby's _here-doc_ and string interpolation to create a block of output:

```
raise <<-EOS

params: #{params.inspect}
@product: #{@product.inspect}

>>
```

### Debug Helper

If you can get to the point of execution where a view template is being rendered, then you can take advantage of the `debug` helper method. It accepts one object as an argument and outputs a nicely formatted YAML representation of the object wrapped in `<pre>` tags. The built-in Rails stylesheet already has styles for the `<pre>` tags for this reason.

For instance, in the form used with the `create` action, I could insert `debug` like this:

```erb
<%= debug @product %>
```

Then when the view is rendered the YAML output will be visible.

But there's a serious problem with adding debug code -- it tends to get left behind and sent into production! One solution I've used is to define this helper in `ApplicationHelper`:

```ruby
  def d(object)
    debug object if Rails.env == "development"
  end
```

Then in the view templates just use it instead of the built-in `debug`:

```erb
<%= d @product %>
```

Now any debug code would be hidden in production. Want to take it a step further? How about this:

```ruby
def d(object)
  if Rails.env == "development"
    debug object
  else
    raise "Debug code running in test & production!"
  end
end
```

Now, assuming that you have coverage of your views in automated tests, debug code will get caught before hitting production!

## A Custom Logger

Let's say you want to debug a more complex process. Maybe it involves several "checkpoints" across multiple methods. Or you want to build an audit trail for actions in your application. You need a custom logger.

It's very easy to create and use thanks to Rails `ActiveSupport::BufferedLogger`. Imagine we want to build an audit log for our application. Start with an initializer `config/initializers/audit_logger.rb`

```ruby
module Kernel
  @@audit_log = ActiveSupport::BufferedLogger.new("log/audit.log")
  def audit(message)
    preamble = "\n[#{caller.first}] at #{Time.now}\nMessage: "
    @@audit_log.add 0, preamble + message.inspect
  end
end
```

Then anywhere in your application call the `audit` method:

```ruby
def create
  @product = Product.new(params[:product])
  audit @product.inspect
  if @product.save
    #...
```

Pop open `log/audit.log` and you'll find messages like this:

{% terminal %}
[/path/to/your/app/controllers/products_controller.rb:18:in `create'] at 2011-07-19 14:12:16 -0700
Message: "#<Product id: nil, title: \"Apples\", price: nil, description: nil, image_url: nil, created_at: nil, updated_at: nil, stock: 0>"
{% endterminal %}

Tweak the formatting to your liking, then add auditing wherever needed in your application!

## Heroku Logs

That's great for development, but what about accessing our logs in production?

The first step is to enable Heroku's "Expanded" logging add-on. From within your project directory, assuming it is already running on Heroku:

{% terminal %}
$ heroku addons:add logging:expanded
{% endterminal %}

### Fetching Logs

Getting logs from Heroku is simple:

{% terminal %}
$ heroku logs --tail
{% endterminal %}

Which will give you output like this:

{% terminal %}
$ heroku logs
2010-09-16T15:13:46-07:00 app[web.1]: Processing PostController#list (for 208.39.138.12 at 2010-09-16 15:13:46) [GET]
2010-09-16T15:13:46-07:00 app[web.1]: Rendering template within layouts/application
2010-09-16T15:13:46-07:00 app[web.1]: Rendering post/list
2010-09-16T15:13:46-07:00 app[web.1]: Rendered includes/_header (0.1ms)
2010-09-16T15:13:46-07:00 app[web.1]: Completed in 74ms (View: 31, DB: 40) | 200 OK [http://myapp.heroku.com/]
2010-09-16T15:13:46-07:00 heroku[router]: GET myapp.heroku.com/posts queue=0 wait=0ms service=1ms bytes=975
2010-09-16T15:13:47-07:00 app[worker.1]: 2 jobs processed at 16.6761 j/s, 0 failed ...
{% endterminal %}

The marker in `[]` corresponds to the dyno generating the message. The log aggregates information from all your dynos and workers, so it's great for tracking down complex interactions between components.

For more extensive discussion, check out the Heroku resource center here: http://devcenter.heroku.com/articles/logging

## Exercises

Grab the Blogger sample project, create a branch, and try out each of the following techniques:

{% terminal %}
$ git clone https://github.com/JumpstartLab/blogger
$ cd blogger
$ git checkout -b my_debugging
$ bundle
{% endterminal %}

1. Add `attr_accessible :title` to the `Article` model. Then create an article through the web interface. Check out the log file from your server and find the warnings, notice the `nil` data in the `INSERT` statement.
2. With that `attr_accessible` still in place, use `warn` statements in the `create` action to output the state of the `Article` object after creation. Find the output in the server log.
3. Now, instead of using `warn`, try using `raise`. Observe what happens when you call `raise` on the object itself. Then add `.inspect` and trigger the `raise` again.
4. Add a call to `debug` in the `show.html.erb` to display the current article.
  * Extra challenge: Write a `d` helper as described above. Add it to your application's layout so every page displays `@article` or `@articles` if they exist.
5. Implement a custom logger and add log entries for each step of the article life-cycle: `create`, `update`, `show`, and `destroy`. Trigger a few of those actions and see that they're output in the audit log.

Once complete, either commit (`git commit`) or get rid of (`git reset --hard`) your changes. You can stay on the debugging branch for the next section.
