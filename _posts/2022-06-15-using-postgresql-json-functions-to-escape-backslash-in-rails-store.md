---
title:  "Using Postgresql JSON functions to escape backslash in Rails store"
date:   2022-06-15 09:00:00
permalink: using-postgresql-json-functions-to-escape-backslash-in-rails-store
---

## Storing Rails store as JSONB

When using [Rails store](https://api.rubyonrails.org/classes/ActiveRecord/Store.html), we can set the data type as [plain text](https://api.rubyonrails.org/classes/ActiveRecord/Store.html) or a structure datatype like json, hstore or jsonb.

Between json and jsonb, depending on the use case, one might be more favorable than the other as the choice of Rails store data type. For 1, json take the text input as it is and saves it as json text, whereas jsonb saves the text input in a “decomposed binary format”, which adds overheads during writing, but reduces overhead when reading since no parsing needed.

In addition, jsonb allows indexing and executing database queries against, even with ActiveRecord. So most of the time, if you’re thinking of using Rails store, you probably would want jsonb. More on the advantages and disadvantages between json and jsonb can be found [here](https://www.postgresql.org/docs/current/datatype-json.html).

In this case, I’m storing it as jsonb because I’m going to query it.

Rails store will save backslash
Creating a simple Rails store like that as [depicted in the docs](https://api.rubyonrails.org/classes/ActiveRecord/Store.html) will generate a string with backslashes to escape the ". Taking this code snippet as an example:

```ruby
class User < ActiveRecord::Base
  store :settings, accessors: [ :height, :weight ], coder: JSON 
end
```

And updating the user like user.update(height: 200, weight: 100) will save the data as "{\"height\": 200, \"weight\": 100}".

This causes a problem when we try to access the values inside via [postgresql JSON functions](https://www.postgresql.org/docs/current/functions-json.html). When we try to do a WHERE users.settings::jsonb->'height' IS NOT NULL, the query will not work as expect and the output will always be null. This is because users.settings is not a proper json, so accessing any keys just would not make sense.

## Escaping Rails store backslashes

The trick here is to first use the #>> annotation to convert the jsonb into text first. This step removes the backslash, but outputs a text.

The resultant text can, in turn, be converted into jsonb via typecasting, like (users.settings#>>'{}')::jsonb.

Thereafter, we can manipulate the jsonb as a proper json and query it with how we would have used the postgresql JSON functions, like this where clause: WHERE (users.settings#>>'{}')::jsonb->'height' IS NOT NULL.

