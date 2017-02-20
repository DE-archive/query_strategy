# Ruby on Rails Query Strategy.

Sometimes folks become confused when trying to perform a complicated query to the SQL-database. In most of cases, it takes lots of time and that output query string looks pretty ugly. In this article I'd like to describe best ways of querying and formatting our code to make it easier for reading and faster for retrieving the data.


Things first, database performance is all about `Indices`. Database tables without indices will degrade in performance as more records get added to them over time. The more records added to the table, the more the database engine will have to look through to find what it’s actually looking for. Adding an index to a table will ensure consistent long-term performance in querying against the table even as many thousands of records get added to it. When you add indices to your tables, the application gains performance without having to alter any model code. For example, imagine you’re fetching the comments associated with a post. If the post `has_many :comments` then the comments table will have a `post_id` column. Adding an index on `post_id` will improve the speed of the query significantly. **NOTICE: need prove. Benchmarking!**


Using Indices is pretty useful and you can ask why an index isn’t added to every column since the performance for searching the table would be improved. Unfortunately, indices don’t come easily. Each insert to the table will incude extra processing to maintain the index. For this reason, indices should only be added to columns that are actually queried in the application.


Consider the comment model in a blogging application. The model has the following fields:

   * post_id 
   * user_id 
   * body 
   * upvotes 

It’s a pretty simple model, isn’t it? However, let us add some indices, using migrations. Let's create indices for both fields `post_id` and `user_id`:

```ruby
class AddIndicesToComment < ActiveRecord::Migration
    def change
        add_index :comments, :post_id
        add_index :comments, :user_id
    end
end
```
or even in one string:
```ruby
    add_index :comments, [:post_id, :user_id]
```
Eventually, there's a preferrable way to use references with initial index for creating connection between them:
```ruby
    add_reference :comments, :post, index: true
```

Another way to speed up the total time of a request is to reduce the number of queries being made. The includes query method is used to eager load the identified child records when the parent object is loaded. Let’s watch through the development log as we interact with a post and its comments in the Rails console:

```
:001> post = Post.first
    Post Load (1.0ms)  SELECT  "posts".* FROM "posts"  ORDER BY "posts"."id" ASC LIMIT 1
=> #<Post id: 1, title: "New Post", upvotes: 0, created_at: "2017-01-21 10:13:13", updated_at: "2017-01-21 10:13:13", user_id: 1>
:002> post.comments
    Comment Load (0.4ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" = $1  [["post_id", 1]]
=> #<ActiveRecord::Associations::CollectionProxy  [#<Comment id: 1, body: "Good Post", upvotes: 0, post_id: 1, created_at: "2017-01-22 16:44:13", updated_at: "2017-01-22 16:44:13", user: id: 15>
```

The two instructions ran two separate queries. But if we use includes in the first query the following happens:

```
:001> post = Post.includes(:comments).first
    Post Load (0.7ms)  SELECT  "posts".* FROM "posts"  ORDER BY "posts"."id" ASC LIMIT 1
    Comment Load (0.4ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" = $1  [["post_id", 1]]
=> #<Post id: 1, title: "New Post", upvotes: 0, created_at: "2017-01-21 10:13:13", updated_at: "2017-01-21 10:13:13", user_id: 1>
```
Only one instruction kicked off two queries, eager fetching both the article and its comments. There’s no performance gain when using includes so far. In fact, each query takes longer to process.



Third way to reduce the total time of a request is to use select or pluck methods for taking only needed fields. Select is a pretty useful method and I guess anyone know how it works, but anyway I’d like to show you what it must return

```
:001> Post.select(:title, :created_at)
    Post Load (0.4ms)  SELECT  "posts"."title", "posts"."created_at"  FROM "posts"
=> #<ActiveRecord::Relation  [#<Post id: nil, title: "New Post", created_at: created_at: "2017-01-21 10:13:13">, <Post id: nil, title: "Ruby Cool", created_at: created_at: "2017-01-21 10:16:16" >]>
```

As far as you can see, select method’s returning `ActiveRecord` collection of instances, in our case Posts.

Ok, now when you know how we can load particular fields from db, let’s look on the `:pluck` method. Use `:pluck` as a shortcut to select one or more attributes without loading a bunch of records just to grab the attributes you want.

```
:001> Post.pluck(:title)
    Post Load (0.2ms)  SELECT  "posts"."title"  FROM "posts"
=> ["New Post", "Ruby Cool"]
```

You may have a question: “What if we want to load more than one field?”

```
:001> Post.pluck(:title, :created_at)
    Post Load (0.2ms)  SELECT  "posts"."title", "posts"."created_at"  FROM "posts"
=> [["New Post", Sat, 21 Jan 2017 10:13:13 UTC +00:00], ["Ruby Cool", Sat, 21 Jan 2017 10:16:16 UTC +00:00]]
```


In that case pluck will return array of selected fields, which, in my opinion, is not very useful and I’ve never used pluck for loading more than one field.

So...the main difference between `:select` and `:pluck` is output, `:select` returns is `ActiveRecord` collection of instances which includes only selected fields, and you can use `ActiveRecord` methods for work with it, `:pluck` method returns simple ruby array of selected field, or array of arrays if you choose more than one filed.

Another thing I want to talk about is Scopes **NOTICE: use highlighing! read .MD docs.**. Probably most of us know about that super useful and elegant feature, however, I saw relatively huge projects that was not using that and having complicated pieces of code instead. So … let me explain what scoping actually is.


Scoping allows you to specify commonly-used queries which can be referenced as method calls on the association objects or models. With these scopes, you can use every method previously covered such as `:where`, `:joins` and `:includes`. All scope methods will return an `ActiveRecord::Relation` object which will allow for further methods to be called on it.




Declaration scopes happen in the Model, just like that

```ruby
class Post < ActiveRecord::Base
    has_many :comments
    belongs_to :user

    scope :nice, -> { where("upvotes > ?", 5) }
end
```
And let’s look what happens when we call it
```
:001> Post.select(:upvotes)
     Post Load (2.4ms)  SELECT  "posts"."upvotes"  FROM "posts"
=> #<ActiveRecord::Relation  [#<Post id: nil, upvotes: 12>, <Post id: nil, upvotes: 0>, <Post id: nil, upvotes: 4>, <Post id: nil, upvotes: 31>]>
:002> Post.nice.select(:upvotes)
    Post Load (0.6ms)  SELECT  "posts"."upvotes"  FROM "posts" WHERE (upvotes > 5)
=> #<ActiveRecord::Relation  [#<Post id: nil, upvotes: 12>, <Post id: nil, upvotes: 31>]>
```

It’s the simplest and the most elegant way of filtering in my opinion.


But what if you want to use that `:scope` for every single call?


I have the answer - use `:default_scope`!
If an object is always going to load its child records, for example posts with included comments, a `:default_scope` can be set up for the particular model. Then every query will be ready to load record's children. Continuing with our previous example, I suppose we always want the comments for a post to be loaded. Instead of having to remember to add `include: :comments` to all finder calls add the following to the Post model:




```ruby
class Post < ActiveRecord::Base
    has_many :comments
    belongs_to :user

    scope :nice, -> { where("upvotes > ?", 5) }

    default_scope include: {comments: :approval}
end
```

After the above has been added to the model, Post.first will include the associated comments. Therefore, if you need that article’s comments, another database query is no longer necessary.

```
:001> Post.first
     Post Load (2.4ms)  SELECT  "posts".*  FROM "posts"  ORDER BY "posts"."id" ASC LIMIT 1
     Comment Load (0.4ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" IN (1)
=> #<Post id: 1, title: "New Post", upvotes: 0, created_at: "2017-01-21 10:13:13", updated_at: "2017-01-21 10:13:13", user_id: 1>
```

Looks much better, isn’t it? With `:default scope` we don’t have to write the ordering in the controller or model methods for every action, that's awesome!


But obviously in some cases we should remove our including comments, you can call unscoped method for that

```ruby
class PostController < AplicationController

    def some_action_where_we_no_need_comments
        @posts = Post.unscoped.all
    end

end
```

And the `:default scope` will be ignored.


In this article I described a small part of all the opportunities of ActiveRecord Querying methods which can help us to write fast and readable code. Hopefully you'd find that helpful!

Best regards,

*Dmitry Gusev* Software developer Dunice Edge

@DmitriyGysev

http://dunice.ru

dmitriy.gusev@duniceedge.com

Skype: edge.gusev
