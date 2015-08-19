---
layout: post
title:  "Black-Box Testing with RSpec and Capybara"
date:   2015-08-19 17:00:00
categories: rspec capybara ruby rails
---

[Capybara][capybara] is a fantastic tool for testing user interactions with our applications. All too often, though, I see tests that use only Capybara's basic features and get littered with confusing and repetitive code. Instead, we can use the page object pattern and Capybara's more advanced features to write readable and maintainable tests. This post will walk you through designing a black box testing framework for your web application that turns a website into a clean, readable Ruby API.

Page Object Pattern
-------------------
Let's say you have a simple page with a short registration form and would like to write some tests for it. Here's the HTML:
{% highlight html %}
<html>
  <body>
    <h2>Register</h2>
    <form action="#">
       <label for='email'>Email</label>
       <input type='text' id='email'><br />
       <label for='password'>Password</label>
       <input type='password' id='password'><br /><br />
       <input type='button' id='save_button' name='Save' value='Save'>
    </form>
    </detail>
  </body>
</html>
{% endhighlight %}

Your first stab at a test might include this:
{% highlight ruby %}
scenario 'registering a user' do
  visit '/register'

  fill_in 'Email', with: 'example@example.com'
  fill_in 'Password', with: 'password'
  click_button 'Save'
end
{% endhighlight %}

This looks pretty simple and readable on its own, and if this were your entire application, this would probably suffice. But think about what other tests will be added next, and a few problems arise. First of all, we're going to have this registration code repeated in several tests, because most features require a registered user. Once that happens, what happens if the path to the registration page changes? Or the "Save" button changes to "Submit"?

Enter page objects. You can define a class for each page - really, each section of a page - and use it to store the logic about how to interact with the page. Reuse then becomes easy, and if there are any changes to the interface, you only need to update one page object instead of many tests.

Here's an example for the page above:
{% highlight ruby %}
class RegistrationPage
  include Capybara::DSL

  URL = '/register'

  def visit
    super(URL)
  end

  def register(email: , password: )
    visit
    fill_in 'Email', with: email
    fill_in 'Password', with: password
    click_button 'Save'
  end
end
{% endhighlight %}

Now your spec looks like:
{% highlight ruby %}
scenario 'registering a user' do
  registration_page = RegistrationPage.new
  registration_page.register(email: 'example@example.com', password: 'password')
end
{% endhighlight %}

If you need to register more users in other tests, it's now much simpler. And if anything changes in the UI the registration page, you only have to update it in one place instead of in many tests.

Building a Page Object Framework for Your Application
-----------------------------------------------------

While building a page object framework for a large web application, we found a lot of repetition between our page objects. For instance, the `visit` method was very similar between most pages, and we also wanted to expand it to ensure we were on that page after visiting it. We also ran into cases where there were sections of many pages that were very similar but not quite identical. We took a different approach to each of these cases.

Basic Page class for all application pages
==========================================

Our solution for the first case was to make a base page class for things that are shared between ALL pages in our application. This includes a `visit` method and logic around the CSS that we use for flash messages and errors throughout the application. You could also include methods to access the header and footer on the page. Here's a sketch of what ours looks like[^1]:
{% highlight ruby %}
class Page
include Capybara::DSL
  attr_reader :url, :url_matcher

  def initialize(url, url_matcher = nil)
    @url = url
    @url_matcher = url_matcher || /#{url}/
  end

  def visit(fail_on_redirect = true)
    super(url)
    if fail_on_redirect
      fail WrongPageVisited, "Expected URL '#{url}', got '#{page.current_url}'" unless displayed?
    end
  end

  def flash_error_css
    'div.flash.error'
  end

  def flash_error
    find(flash_error_css)
  end

  def has_flash_error?(error)
    has_css?(flash_error_css, :text => error)
  end

  def has_no_flash_error?(error)
    has_no_css?(flash_error_css, :text => error)
  end
end
{% endhighlight %}

You can use this base `Page` object throughout your application. Here's your new `RegistrationPage`:
{% highlight ruby %}
class RegistrationPage < Page
  def initialize
    super('/register', /register/)
  end

  def register(email: , password: )
    visit
    fill_in 'Email', with: email
    fill_in 'Password', with: password
    click_button 'Save'
  end
end
{% endhighlight %}

Sharing similar, but not identical, page sections
=================================================

If you have something that is identical on several pages - such as a header or footer - it should be its own page object that gets injected into other page objects as necessary (or used on its own). But what if you have very similar sections that aren't _quite_ identical? We ran into this with things we dubbed Forms and Tables. Throughout our application, there are pages where you can input data and pages where you can view that data. Making the page objects for these forms and tables was repetitive, but putting them into `Page` or making them into page objects of their own didn't quite work because of the differences (and we didn't want to bloat `Page`).

Our solution was to use some metaprogramming and dependency injection. We wanted this functionality:

{% highlight ruby %}
class RegistrationPage < Page
  include Forwardable

  attr_reader :form

  def initialize
    super('/register', /register/)

    form_fields = {email: 'Email', password: 'Password'}
    form = Form.new(form_fields)
    def_delegators :form, *form.field_method_names
  end

  def register(email: , password: )
    visit
    form.fill_in_form(email: email, password: password)
  end
end
{% endhighlight %}

In addition to making it much easier to spin up new pages with forms on them, this adds a lot more functionality to `RegistrationPage`. I can now fill in fields manually if I want to do more detailed testing:
{% highlight ruby %}
scenario "changing a registration field's value" do
  registration_page = RegistrationPage.new
  registration_page.visit
  registration_page.first_name_field.set('John')
  registration_page.first_name_field.set('Jane')
  expect(registration_page.first_name_field.value).to eq('Jane')
end
{% endhighlight %}

Here's a simplified version of how `Form` is implemented:
{% highlight ruby %}
class Form
  include Capybara::DSL

  attr_accessor :fields

  def initialize(fields)
    @fields = fields
    define_field_methods
  end

  def fill_in_form(fields)
    fields.each do |field, value|
      f = send("#{field}_field")
      f.set(value) || Array(value).each { |val| f.select(val) }
    end
  end

  private

  def define_field_methods
    fields.each do |field, field_string|
      field_method_name = define_field_name(field)
      define_field_method(field_method_name, field_string)
    end
  end

  def define_field_method(field_method_name, field_string)
    field_string = field_string.chomp(':')
    self.class.send(:define_method, field_method_name) do
      find_field(field_string)
    end
  end
end
{% endhighlight %}

As you make page objects for the various pages and page sections in your application, consider making classes like this for parts that seem repetitive.

Delegating to Capybara elements
===============================
I'll leave you with one more tip to keep in mind when designing page objects for your new page object framework. For certain sections of pages, it may be handy to have them as a `Capybara::Element` so that you can continue to use Capybara's methods like `find` on them (to do things like find other elements within them), but you may also want to separate them out into "page" (partial page) objects and define other methods on them. We've used `DelegateClass` for this purpose. Here's a short example from our app, for an "Edit" dialog popup.
{% highlight ruby %}
class EditDialog < DelegateClass(Capybara::Node::Element)
  def initialize
    super(Capybara.page.find('div.edit-dialog'))
  end

  def assign_installer(name)
    find_link('Assign Installer').click
    find_field('Assign To').select(name)
    find_button('Assign').click
  end
end
{% endhighlight %}

Conclusion and Further Reading
------------------------------
You now have the tools you need to turn your website under test into a clean, readable Ruby API. Your tests will be more concise, more readable, and easier to change. Here's an example of a real integration test of our application using page objects:
{% highlight ruby %}
scenario 'installers can move inventory to and from warehouses' do
  login_page.authenticate(as: 'installer1')

  inventory_page.visit
  inventory_page.open_add_to_my_inventory_tab

  inventory_page.search_for_device('0000001')
  inventory_page.add_device
  inventory_page.submit

  expect(inventory_page).to have_flash_message('Successfully updated selected inventory.')

  inventory_page.open_detail_tab

  expect(inventory_page).to have_device_count(1)

  inventory_page.device('0000001').check
  inventory_page.change_location('Test Warehouse')

  expect(inventory_page).to have_device_count(0)
end
{% endhighlight %}
Clear and concise testing!

Further reading and references
==============================

* Martin Fowler's [PageObject][fowler] post.
* RailsConf presentation by Eduardo Gutierrez: [slides][eddie slides] and [video][eddie video] - he doesn't advocate for page objects, but he has great Capybara tips and tricks.
* [This post][ferris] by Joe Ferris on writing reliable Capybara tests.
* [This post][jnicklas] by Jonas Nicklas (creator of Capybara) on tricks for cleaning up your Capybara tests.


Notes
=====

[^1]: A note on getting the most out of Capybara: see the `has_flash_error?` and `has_no_flash_error?` methods in the code block? At first glance, you might wonder why both are necessary. It has to do with the way that Capybara waits for elements to appear on a page. `has_css?` will return `true` as soon as the specified CSS appears, but it will only return `false` after waiting the default wait time configured in Capybara. `has_no_css?`, on the other hand, will return `true` as soon as the CSS is not there, and it will only return `false` after waiting and still not having the CSS appear. If you want to assert that something is _not_ on the page, you should always use `has_no_css?`, or your test will always wait the default wait time before passing.

[capybara]: https://github.com/jnicklas/capybara
[fowler]:   http://martinfowler.com/bliki/PageObject.html
[eddie slides]: https://speakerdeck.com/ecbypi/ambitious-capybara
[eddie video]: http://confreaks.tv/videos/railsconf2015-ambitious-capybara
[ferris]: https://robots.thoughtbot.com/write-reliable-asynchronous-integration-tests-with-capybara
[jnicklas]: http://www.elabs.se/blog/51-simple-tricks-to-clean-up-your-capybara-tests


