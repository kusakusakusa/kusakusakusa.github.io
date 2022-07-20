---
title:  "How To Edit A Rake Task From A Gem"
date:   2022-05-04 09:00:00
permalink: how-to-edit-a-rake-task-from-a-gem
---

Apart from having to contribute to the open source project of a gem that you’re using in your Ruby On Rails application in order to edit a rake task in there, you can simply use Rake::Task#enhance to add some codes before and/or after the rake task. Unfortunately, Rake::Task#enhance does not allow adding anything in the middle of the rake task 😢

## The Endgame

Eventually, you just need to run the same rake task command rake my:task, and the pre and post actions will be executed as well

## How To Use Enhance On rake task

`Rake::Task#enhance` is execute on the rake task that you intend to “enhance”, so to execute it would look like Rake::Task['my:task'].enhance.

It takes in a &block argument that would run after the rake task has completed. I love that you can add binding.pry in this block of code to debug the post rake task action that you intend to enhance too.

Here is my practical example using [Rake::Task#enhance](https://ruby-doc.org/stdlib-trunk/libdoc/rake/rdoc/Rake/Task.html#method-i-enhance). I use the [i18n-js](https://github.com/fnando/i18n-js) gem whose function is to take all the yaml locale files in rails and translate them into javascript variables in a javascript file. The application can then do the translation on the frontend in javascript without having to reload to render the page from server side in order the get the translation from the backend.

The problem with it is that the translation javascript file that it generates does not include a fingerprint (last time I read the docs, there does not seem to have an option to add the fingerprint too). This causes problem for us when we update the content in the translation.js file since we use CDNs to cache our assets. When we update the translation.js file, the old cached version is served because the name of the file is the same.

The code snippet below enhances the rake task by:

1. Generating the translation.js file first
2. Removing the previous translation-XXfingerprintXX.js files if any
3. Add the current timestamp as the fingerprint for the newly generated js file in step 1.

```ruby
Rake::Task['i18n:js:export'].enhance do
   puts "Removing previous translations.js files"
   Dir.glob(Rails.root.join('public/javascripts/translations-*.js')).each do |file|
     File.delete(file)
   end
 puts "Adding fingerprint to public/javascripts/translations.js…"
   File.rename(
     Rails.root.join('public/javascripts/translations.js'),
     Rails.root.join("public/javascripts/translations-#{Time.current.to_i}.js")
   )
end
```

## How To Add Code Before The rake task

Above, we cover how to run the task after the rake task has run. To run something before that, you would need to pass the code as an argument.

```ruby
Rake::Task['my:task'].enhance(puts("Start my:task")) do
  puts("End my:task")
  puts("Start my:new:task")
  Rake::Task['my:new:task'].invoke
  puts("End my:new:task")
end
```

## Other Examples

If you use capistrano to do deployment, you can refer to their github source code for the after macro.

It uses enhance to synchronous order of the rake tasks to run during deployment.

## Improvements

Since we are providing a &block argument, I would have expected to be able to use the yield keyword to control when to execute the rake task, which is the commonly used in ruby or rails. So for example,

```ruby
Rake::Task.enhance do
  puts "Start"
  yield
  puts "End"
end
```

I believe this will improve familiarity of the code to fellow rubyists! Good old convention over configuration philosophy!

Note to self: There might be some limitations/problems executing a rake task this way; explore first and maybe propose in rails github 🥶

