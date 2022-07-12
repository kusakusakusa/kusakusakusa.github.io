---
title:  "How To Setup Headless Chrome Browser For Rails RSpec Feature Test With Selenium And Capybara"
date:   2020-12-16 09:00:00
permalink: how-to-setup-headless-chrome-browser-for-rails-rspec-feature-test-with-selenium-and-capybara
---

When working on feature tests on a non headless browser, the automated browser will tend to take control over our display and we tend to lose focus. That means while the test is ongoing, we cannot do anything but to wait for the test to run its course. And that sucks.

Headless chrome to the rescue!

[Since April in 2017, Chrome ships with a headless version](https://developer.chrome.com/blog/headless-chrome/). It allows running the browser without a GUI. That is just perfect for automated feature testing without having to lose control over your machine.

Fast forward to 2020 (actually it’s been around for quite a while but I just couldn’t set it up properly, until recently), you can setup headless chrome to run feature tests in your Rails application with almost no custom code that are suggested all over stackoverflow. I will go through how using Capybara and Selenium.

# Install Latest Capybara and Webdrivers gems

It is important to make sure you are using the latest version of gems. Learn how to find the latest ruby gem version here.

```ruby
# Gemfile
group :test do
  gem 'capybara', '~> 3.34'
  gem 'webdrivers', '~> 4.0'
end
```

## Setup the Configuration for Selenium Headless Chrome

Setup this configuration in your rails_helper.rb file.

```ruby
# rails_helper.rb

Capybara.javascript_driver = :selenium_chrome_headless
# Capybara.javascript_driver = :selenium_chrome
```

Line 3 specifies to use the default selenium chrome headless browser as the driver to run your feature tests. This driver comes with the latest capybara gem as one of the [default drivers](https://github.com/teamcapybara/capybara/blob/f43b6499ecb2721dd97b63a7fe31ff2518b89b4c/lib/capybara/registrations/drivers.rb).

Line 4 specifies to use the default selenium chrome browser to run your tests. This setting will run browser in the foreground and you will lose control of your machine. It is commented out, but you probably might need it when you are debugging your tests, so I will leave it here. To use it, simply uncomment it and comment out line 3.

## And there you have it!

That is all the configuration you need. The whole setup is already incorporated into the latest gems so you do not need to add in any more custom code.

