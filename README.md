# Ruby on Rails Query Strategy.
Hi there, my name is Dmitry Gusev and i’m a RoR Developer.
Sometimes folks become confused when they try to take something from db, it takes a lot of time and that query string looks very ugly. In this article I want to describe the best ways of querying and formatting our code for making it easier for reading and faster for querying.
First thing I want to talk is Indices. Database tables without indices will degrade in performance as more records get added to them over time. The more records added to the table, the more the database engine will have to look through to find what it’s looking for. Adding an index to a table will ensure consistent long-term performance in querying against the table even as many thousands of records get added to it.
When you add indices to your tables, the application gains performance without having to alter any model code.
For example, imagine you’re fetching the comments associated with a post. If the post
```ruby
has_many :comments
```
then the comments table will have a post_id column. Adding an index on post_id will improve the speed of the query significantly.
Using Indices is pretty usefull and you can ask why an index isn’t added to every column since the performance for searching the table would be improved. Unfortunately, indices don’t come easily. Each insert to the table will incur extra processing to maintain the index. For this reason, indices should only be added to columns that are actually queried in the application.
Consider the comment model in a blogging application. The model has the following fields:
* post_id
* user_id
* body
* upvotes

It’s a pretty simple model, isn’t it. But let us add indeces, we can do it via migrations. Let create the indeces for field post_id and user_id, we can make it for every field
```ruby
AddIndecesToComment < ActiveRecord:Migration
    def change
        add_index :comments, :post_id
        add_index :comments, :user_id
    end
end
```
and in one string
```ruby
AddIndecesToComment < ActiveRecord:Migration
    def change
        add_index :comments, [:post_id, :user_id]
    end
end
```
but I prefer to use references with initial index for creating connection between them
```ruby
AddIndecesToComment < ActiveRecord:Migration
    def change
        add_reference :comments, :post, index: true
    end
end
```
Another way to improve the total time of a request is to reduce the number of queries being made. The includes query method is used to eager load the identified child records when the parent object is loaded. Let’s watch the development log as we interact with a post and its comments in the Rails console:
