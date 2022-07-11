---
title:  "Reopen And Add Methods To Models In Ruby Gems"
date:   2019-10-27 09:00:00
permalink: reopen-and-add-methods-to-models-in-ruby-gems
---

This is a documentation on how to add class and instance methods to models that exist in ruby gems. Often, there is a need to add methods to models that are created in ruby gems.

In a recent project that I am working on, I found a particular need for adding images to a tagging gem. The purpose of the gem is for taxonomy and the final UI draft has different images allocated to each of the category (or should I say tag). We decided to have the rails backend handle the image and tag association. Hence, the ideal way to handle this would be to modify the models in the tagging gem to hold its image as well upon creation.

## Project Specifics

The tagging gem that I am using is, contrary to the more popular and senior `ActsAsTaggableOn` gem, the `Gutentag` gem.

The reason I use the latter instead of the former is because the former does not support the new `ActiveRecord` 6 when I was working on the project. It returns erroneous results and throws error due to deprecated `ActiveModel` method in its normal usage for example.

The alternative I found is `Gutentag`. It support `Rails 6` and the contributors are actively resolving issues, keeping its issues count at 0 at the time of writing. I found it reliable and it does its main job well, which is to provide the tagging module.

The only thing it lacks for this particular project is an image to associate with for each tag. Here is where I would need to hack it.

I want to add image to each tag using `ActiveStorage` via `has_on_attached` method, and also a custom instance method that will return the tag’s name and image url.

## The Rationale

The way I am doing it is to create a module that defines the relevant methods, and have the `Gutentag::Tag` model include this custom module. I will include it during the initialization phase. This will require some workarounds because of we are accessing the `ActiveStorage` and `ActiveModel/ActiveRecord` railitie sduring the initialization phase where these railities are not loaded yet.

## The Extension Module

Kudos to [this answer on stackoverflow](https://stackoverflow.com/questions/17608006/how-to-reopen-a-class-in-gems/48665268#48665268), define the extension module as such:

```ruby
# lib/extensions/gutentag.rb
# frozen_string_literal: true

module Extensions
  module Gutentag
    extend ActiveSupport::Concern
    
    included do
      has_one_attached :image
    end

    def json_attributes
      custom_attributes = attributes.dup
      custom_attributes.delete 'created_at'
      custom_attributes.delete 'updated_at'
      custom_attributes.delete 'taggings_count'
      custom_attributes.delete 'id'

      #  add image path base on service used
      if Rails.env.test? || Rails.env.development?
        ActiveStorage::Current.set(host: 'http://localhost:3000') do
          custom_attributes['image'] = self.image.attached? ? self.image.service_url : nil
        end
      else
        custom_attributes['image'] = self.image.attached? ? self.image.service_url : nil
      end

      custom_attributes
    end
  end
end
```

I modified it slightly with the use of `ActiveSupport::Concern` to do it the `Rails 6` way. This helps to resolve module dependencies gracefully.

In this extension, I attached an image to the module using `ActiveStorage`‘s `has_one_attached` class method, which will ultimately be applied to the `Gutentag::Tag` model.

I also defined the instance method json_attributes which will return only the name and the image url in the resultant tag when called. It is used in the api response when frontend clients are retrieving the list of tags for example.

## The Initialization

The code will be added to the original `Gutentag` initializer file under `config/initializers/gutentag.rb`.

```ruby
# frozen_string_literal: true

require 'extensions/gutentag'

Gutentag.normaliser = lambda { |value| value.to_s }

Rails.application.config.to_prepare do
  begin
    if ActiveRecord::Base.connection.table_exists?(:gutentag_tags)
      Gutentag::Tag.include Extensions::Gutentag
    end
  rescue ActiveRecord::NoDatabaseError
  end
end
```

The extension file is imported in line 3.

Line 5 is one of the provided original Gutentag configuration option. This is specific to my project and is trivial in relation to the topic of this article. I am leaving it here to show other Gutentag configuration changes will co-exist with this custom module of mine.

Line 10 is the main line of code to execute. It will add the module to the `Gutentag::Tag` model which is defined inside the source code of the `Gutentag` gem. However, as you can see, it is wrapped in a number of codes. Not doing so will result in errors.

Here is why.

As we are going to involve the `ActiveRecord` and `ActiveSupport` railities, which have not been initialized yet during the default rails initialization phase, we need to ensure we run the code after they have been loaded.

Rails has 5 initialization events. The first initialization event to fire off after all railities are loaded is `to_prepare`, hence we define the code after that happens inside its block.

Since we are interacting with a `ActiveRecord` model, during the initialization phase, it is possible that the table has not been created. In other words, the Gutentag tables migration has not been executed, resulting in errors about the table not existing. An if conditional check is done to prevent this error.

I am not handling the else condition as under normal circumstances, after the proper migration has been executed, this will not happen. A possible scenario that this would happen is during rake tasks to create or migrate the database, either of which does not use the new methods at all.

A non-existing table is not the only thing we have to guard when dealing with Railities during the initialization process. A non-existing database is also a probable scenario that may occur. An example is during the `rails db:create` step. Hence, we rescue the `ActiveRecord::NoDatabaseError` error to silence the error. As this is often the only scenario that will happen, I will not handle the exception in the rescue block.

## Usage

Now we can use it in our application. For instance, I can seed some default tags with images attached to them as shown:

```ruby
# db/seeds.rb
# frozen_string_literal: true

p 'Creating Tags'
[
  'Luffy',
  'Zoro',
  'Usopp',
  'Sanji',
  'Nami',
  'Chopper',
  'Robin',
  'Frankie',
  'Brooks',
].each do |name|
  tag = Gutentag::Tag.create!(name: name)
  tag.image.attach(io: File.open("#{Rails.root.join('app', 'assets', 'images')}/#{name}_avatar.jpg"), filename: "#{name}_image.jpg")
  tag.save!
end
p 'Tags created'
Then in my api response for listing the tags, I can use the json_attributes method as such:

# app/controllers/api/v1/tags_controller.rb
# frozen_string_literal: true

module Api
  module V1
    class TagsController < Api::BaseController
      def index
        @tags = Gutentag::Tag.order(:name)
      end
    end
  end
end

# app/views/api/v1/tags/index.json.jbuilder
json.tags do
  json.array! @tags do |tag|
    json.merge! tag.json_attributes
  end
end
```
