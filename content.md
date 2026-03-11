# Photogram Industrial Part 2: Associations, validations, and sample data

## Walkthrough video

**Please note**, the video is from a previous iteration of the project, so there are some differences:

- Anything contained in the project "README" is now contained in this Lesson

Did you read the differences above? Good! Then [here is a walkthrough video for this project.](https://share.descript.com/view/P3PGeVSVtMW )

**As you watch the video, pause it frequently, read the associated text, and type out the code.**

## Getting started

Let's continue with the `photogram-industrial`, keeping in mind a rough target to work towards:

[photogram-industrial.matchthetarget.com](https://photogram-industrial.matchthetarget.com/)

So navigate to `github.com/codespaces` (or reopen the previous lesson and use the "Load assignment" button) and reopen your `photogram-industrial` project codespace to continue building on what you accomplished in _Photogram Industrial Part 1_.

For this lesson, we'll work on a new git branch called `<your-initials>-photogram-industrial` (e.g., `rb-photogram-industrial`). We can make feature branches off of this branch as we go along.

<div class="alert alert-info">

Here are some [video instructions](https://share.descript.com/view/RLP4apAu5pp) for opening pull requests on GitHub. Follow along there to create a branch and open a _PR_. You'll see this video again in a later lesson as well.
</div>

## Comments

We can generate the rest of our tables with a few more commands. Let's start with `comments`:

```
rails generate scaffold comment author:references photo:references body:text
```

For the `comments` migration file, we need to point the `author_id` foreign key to the `users` table (since we're using `author` instead of `user`), and we should constrain the `body` column to not be empty:

```ruby{4,5,6}
class CreateComments < ActiveRecord::Migration[8.0]
  def change
    create_table :comments do |t|
      t.references :author, null: false, foreign_key: { to_table: :users }
      t.references :photo, null: false, foreign_key: true, index: true
      t.text :body, null: false

      t.timestamps
    end
  end
end
```
{: filename="db/migrate/<date-time-of-migration>_create_comments.rb" }

We also need to note our departure from convention in the `Comment` model:

```ruby{2}
class Comment < ApplicationRecord
  belongs_to :author, class_name: "User"
  belongs_to :photo
end
```
{: filename="app/models/comment.rb" }

And we should add the `has_many` into `User`:

```ruby{8}
class User < ApplicationRecord
  # Include default devise modules. Others available are:
  # :confirmable, :lockable, :timeoutable, :trackable and :omniauthable
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable

  has_many :own_photos, class_name: "Photo", foreign_key: "owner_id"
  has_many :comments, foreign_key: "author_id"
end
```
{: filename="app/models/user.rb" }

And `Photo`:

```ruby{3}
class Photo < ApplicationRecord
  belongs_to :owner, class_name: "User"
  has_many :comments
end
```
{: filename="app/models/photo.rb" }

Now that you have `comments` in good shape, you can `rake db:migrate`.

Next, run a `git status`. This should show a list of untracked files that were added or changed by our previous steps.

We can add and commit all of them to our branch with

```
% git add -A
% git commit -m "Generated comments"
```

And push the commits (along with our new branch!) to Github for safekeeping with:

```
% git push
```

But we get a fatal git error! RTEM! The _first_ time we push from a branch we need to run:

```
% git push --set-upstream origin branch-name
```

Then every subsequent time we can push with just `git push`.

## FollowRequests

For `follow_requests`, we can generate the migration file and scaffold with:

```
rails generate scaffold follow_request recipient:references sender:references status
```

And starting with the `FollowRequest` model, we'll need to note our departure from conventional names with the `recipient` and `sender` ID columns (both referencing the `users` table):

```ruby{2,3}
class FollowRequest < ApplicationRecord
  belongs_to :recipient, class_name: "User"
  belongs_to :sender, class_name: "User"
end
```
{: filename="app/models/follow_request.rb" }

<div class="alert alert-info">

If you're rusty, create an Idea in the [Association Accessor app](https://association-accessors.firstdraft.com/), add the five models, and plan out the association accessor methods you want to add there. [See the last six minutes beginning at 23:30 of this walkthrough video](https://share.descript.com/view/wy5mgzsL2WX) for a refresher on that tool.
</div>

How about the migration file?

```ruby{4:(43-77),5:(40-73),6:(23-42)}
class CreateFollowRequests < ActiveRecord::Migration[8.0]
  def change
    create_table :follow_requests do |t|
      t.references :recipient, null: false, foreign_key: { to_table: :users }, index: true
      t.references :sender, null: false, foreign_key: { to_table: :users }, index: true
      t.string :status, default: "pending"

      t.timestamps
    end
  end
end
```
{: filename="db/migrate/<date-time-of-migration>_create_follow_requests.rb" }

We needed to indicate the departure from convention with `to_table: :users`, and we do want to be able to lookup follow requests by the user's ID, so we leave the default behavior of `index: true` for the foreign keys. We also want to set a default `:status` of `"pending"`, which will then be updated to either `"accepted"` or `"rejected"`.

With that, we can do another

- `rake db:migrate`, then
- `git add -A`, then
- `git commit -m "Generated follow requests"`, and finally
- `git push`.

## Likes

The last resource to generate is `likes`:

```
rails generate scaffold like fan:references photo:references
```

And beginning with the migration:

```ruby{4:(37-70)}
class CreateLikes < ActiveRecord::Migration[8.0]
  def change
    create_table :likes do |t|
      t.references :fan, null: false, foreign_key: { to_table: :users }, index: true
      t.references :photo, null: false, foreign_key: true, index: true

      t.timestamps
    end
  end
end
```
{: filename="db/migrate/<date-time-of-migration>_create_likes.rb" }

In this case, we just need to make one change to point `fan_id` to the `users` table.

Correspondingly in our model:

```ruby{2}
class Like < ApplicationRecord
  belongs_to :fan, class_name: "User"
  belongs_to :photo
end
```
{: filename="app/models/like.rb" }

And that looks good, so migrate, commit, and push like before.

Now that you've pushed all this code to Github, [you can open a pull request for the new branch](https://share.descript.com/view/RLP4apAu5pp). That's a good thing to do early, then you can receive comments on your code and continue to push your commits and have them added to that PR!

## Run `grade`

At this point, try and run `grade` to see if the first few tests are passing. If you set up your migrations and models exactly as described up to this point, then you should see that many of the tests are passing. There is still much to do, but we're making progress!

## Association accessors

We have our resources in place. Now it's time to flesh out the business logic in our models.

That is always my next step. Associations and validations. The direct associations have a good start. We were using the `references` datatype in the `scaffold` generator, which put in most of the `belongs_to` declarations corresponding to foreign key columns for us. We were also diligent about adding the other side of those 1-N associations with the `has_many` declarations we added whenever we saw a `belongs_to`. Nevertheless, let's go through all of our `belongs_to` / `has_many` and see some more useful things to add.

### Direct associations: `belongs_to`

In previous projects, we changed a default setting of Rails: we made it so that `belongs_to` allows foreign key columns to be blank unless you explicitly add the option `required: true`.

In standard Rails applications, the default is opposite: `belongs_to` adds an _automatic validation_ to foreign key columns enforcing the presence of a valid value unless you explicitly add the option `optional: true`.

So: if you decided to _remove_ the `null: false` database constraint from any of your foreign key columns in the migration file (e.g., change `t.references :recipient, null: false ...` to `t.references :recipient, null: true ...`), then you should also _add_ the `optional: true` option to the corresponding `belongs_to` association accessor.

So remember — if you're ever in the situation of:

 - you're trying to save a record
 - the save is failing
 - you're doing the standard debugging technique of printing out `zebra.errors.full_messages`
 - you're seeing an inexplicable validation error message saying that a foreign key column is blank
 - now you know where the validation is coming from: `belongs_to` adds it automatically
 - so figure out why you're not providing a valid foreign key (usually it is because the parent object failed to save for its own validation reasons)

---

#### `counter_cache`

A handy option to add to `belongs_to` is `:counter_cache`. [Read about it](https://guides.rubyonrails.org/association_basics.html#options-for-belongs-to-counter-cache) and add it where you think it's appropriate. Fortunately, we already have columns ready and waiting.

<aside markdown="1">

If you need even fancier counter caches in future projects, check out the [counter_culture gem](https://github.com/magnusvk/counter_culture).
</aside>

Let's start with the `Like` model:

```ruby
class Like < ApplicationRecord
  belongs_to :fan, class_name: "User"
  belongs_to :photo
end
```
{: filename="app/models/like.rb" }

At the moment, it's a bit difficult to decide where to add the `:counter_cache` option, because our model isn't annotated with column names. We could go into the `db/schema.rb` file and see the record of our database. (Note: **_NEVER_** edit the `schema.rb` file. It is auto-generated whenever you run `rake db:migrate`.)

Instead of constantly referencing this file, let's install the `annotaterb` gem. You can read about the [annotaterb gem in its Github README](https://github.com/drwl/annotaterb?tab=readme-ov-file#annotaterb).

<div class="alert alert-danger" markdown="1">

The annotaterb gem was already added in this iteration of the project, so you don't need to actually take the steps shown here and in the video. But you should watch and read along!
</div>

Typically, we add the gem to the `:development` group in our `Gemfile`. (There's no use for this in production or test, so why waste memory loading it there?)

```ruby
# ...
group :development do
  gem 'annotaterb'
# ...
```
{: filename="Gemfile" }

Then run `bundle install` (just `bundle` for short).

While we're in the `Gemfile`, let's also add the `strip_attributes` gem, which automatically strips leading and trailing whitespace from model attributes before validation:

```ruby
gem "strip_attributes"
```
{: filename="Gemfile" }

Run `bundle install` again after adding it.

With the `annotaterb` gem installed, we can now run the terminal command:

```
annotaterb models
```

Alternatively, you can just run:

```
rails g annotate_rb:install
```

Similar to how we finished installing Devise. And this will create a rake task file `lib/tasks/annotate_rb.rake`, so that anytime we run `rake db:migrate` the annotation is run automatically.

Now at the top of our `Like` model in `app/models/like.rb`, we should see the helpful annotations:

```ruby
# == Schema Information
#
# Table name: likes
#
#  id         :bigint           not null, primary key
#  created_at :datetime         not null
#  updated_at :datetime         not null
#  fan_id     :bigint           not null
#  photo_id   :bigint           not null
```

Wonderful. Now let's get back to what we were doing before: adding the `:counter_cache` to any `belongs_to` whenever I'm trying to keep track of the number of children objects that I've got.

Let's start with the `Comment` model

```ruby
# == Schema Information
#
# Table name: comments
#
#  id         :bigint           not null, primary key
#  body       :text
#  created_at :datetime         not null
#  updated_at :datetime         not null
#  author_id  :bigint           not null
#  photo_id   :bigint           not null
# ...

class Comment < ApplicationRecord
  belongs_to :author, class_name: "User"
  belongs_to :photo
end
```
{: filename="app/models/comment.rb" }

So anytime a comment is created, do I want to update the `author` with the count of the comments? We were planning to do that because we added a `comments_count` column in the `users` table:

```ruby{6}
# == Schema Information
#
# Table name: users
#
#  id                     :bigint           not null, primary key
#  comments_count         :integer          default(0)
#  email                  :citext           default(""), not null
# ...
```
{: filename="app/models/user.rb" }

That means we can just add this option on `belongs_to: author`:

```ruby{3:(41-61)}
# ...
class Comment < ApplicationRecord
  belongs_to :author, class_name: "User", counter_cache: true
  belongs_to :photo, counter_cache: true
end
```
{: filename="app/models/comment.rb" }

For this to work, you have to have a column in the `user` table called _exactly_ `comments_count`. We also want to keep count on the `belongs_to :photo`, so add the `counter_cache: true` there as well.

Now in the `Like` model:

```ruby{3-4}
# ...
class Like < ApplicationRecord
  belongs_to :fan, class_name: "User", counter_cache: true
  belongs_to :photo, counter_cache: true
end
```
{: filename="app/models/like.rb" }

Because both `User` and `Photo` have `likes_count` columns.

And `Photo`:

```ruby{3}
# ...
class Photo < ApplicationRecord
  belongs_to :owner, class_name: "User", counter_cache: true
  has_many :comments
end
```
{: filename="app/models/photo.rb" }

We already included a `photos_count` column in our `users` table from the initial generation in Part 1, so this counter cache should work right away.

_Have you been git committing along the way?!_

---

### Direct associations: `has_many`

Now for each `belongs_to`, there should be an inverse `has_many`. Let's go through and make sure we have all of these inverse 1-N associations.

Let's check off the `belongs_to` statements one by one. In the `Comment` model (`app/models/comment.rb`), we see a `belongs_to :author ...`. That means, for `User` we need:

```ruby{3}
class User < ApplicationRecord
  # ...
  has_many :comments, foreign_key: :author_id
end
```
{: filename="app/models/user.rb" }

We need to specify that the foreign key column in comments is not `user_id`, but rather `author_id`. We don't need to say `class_name: "Comment"`, because it matches with the method name `has_many :comments`.

The `Comment` model also contains a `belongs_to :photo ...`. That means, for `Photo` we need:

```ruby{3}
class Photo < ApplicationRecord
  # ...
  has_many :comments
end
```
{: filename="app/models/photo.rb" }

Nothing special there. The foreign key in the `comments` table is `photo_id` (check it with the annotation at the top of `app/models/comment.rb`), and the class name matches the method again.

In `FollowRequest`, we have `belongs_to :recipient` and `belongs_to :sender`, so we need two corresponding associations in `User`:

```ruby{3-4}
class User < ApplicationRecord
  # ...
  has_many :follow_requests, foreign_key: :sender_id, class_name: "FollowRequest"
  has_many :follow_requests, foreign_key: :recipient_id, class_name: "FollowRequest"
end
```
{: filename="app/models/user.rb" }

But we can't have two associations with the same name! So what do we do?

How about this naming:

```ruby{1:(12-32),2:(12-36)}
  has_many :sent_follow_requests, foreign_key: :sender_id, class_name: "FollowRequest"
  has_many :received_follow_requests, foreign_key: :recipient_id, class_name: "FollowRequest"
```

On to `Like`, we find `belongs_to :fan` and `belongs_to :photo`. So, we need corresponding `has_many`s in `User` and `Photo`. First for `User`:

```ruby{3}
class User < ApplicationRecord
  # ...
  has_many :likes, foreign_key: :fan_id
end
```
{: filename="app/models/user.rb" }

Then for `Photo`:

```ruby{3}
class Photo < ApplicationRecord
  # ...
  has_many :likes
end
```
{: filename="app/models/photo.rb" }

And, while we're in that `Photo` model, we see that we need a `has_many` for the `User` model to go with the `belongs_to: owner`. So back in the `User` model:

```ruby{3}
class User < ApplicationRecord
  # ...
  has_many :own_photos, foreign_key: :owner_id, class_name: "Photo"
end
```
{: filename="app/models/user.rb" }

Since the `User` is going to have a few different associations to `Photo`, we gave this one a distinct method name and pointed it to the correct table.

That was a lot of typing! I didn't even check any of the associations in the `rails console`. This is the importance of our `sample_data` task. When we write and run the sample data, any mistakes or holes in our associations will become clear.

Be sure to commit and push after all that work!

### Indirect associations

Before we write any sample data tasks, we should finish trying to write out our indirect associations.

For a given `User`, I need to know who they're following so that I can get the photos posted by those people ("leaders"). And then have another method ("discover") to find out the photos that the people that I'm following have liked.

Let's get started.

Our first N-N is `fans` to `photos` through `likes`. So how do we get there?

We need to go the `Photo` model, and add a `has_many` association for the `fans` of each photo that goes through the `likes` on the photo to the `User` model:

```ruby{4}
class Photo < ApplicationRecord
  # ...
  has_many :likes
  has_many :fans, through: :likes
end
```
{: filename="app/models/photo.rb" }

When we call `has_many :likes`, we have to remember that the method is referring to the model `Like`. If we examine `app/models/like.rb`, it has the association `belongs_to :fan ...`, which returns `ActiveRecord` instances of `User` from our database.

Because we named our new association in `Photo` the plural `:fans`, we don't need to write: `has_many :fans, through: :likes, source: :fan`. We have the conventional name, so Rails will figure it out without the `source:` keyword.

Now, if there's a path from a `Photo` to the `fans`, then it's also possible to go from the `fans` to the `Photo`. Am I ever going to want to go from the user to all of the photos that they have liked? Yes! So let's add this inverse indirect association to `User`:

```ruby{10}
class User < ApplicationRecord
  # ...
  has_many :sent_follow_requests, foreign_key: :sender_id, class_name: "FollowRequest"
  has_many :received_follow_requests, foreign_key: :recipient_id, class_name: "FollowRequest"

  has_many :likes, foreign_key: :fan_id

  has_many :own_photos, foreign_key: :owner_id, class_name: "Photo"

  has_many :liked_photos, through: :likes, source: :photo
end
```
{: filename="app/models/user.rb" }

We get from a user to their collection of likes with the `has_many :likes` we already defined, which brings us to the `Like` model. And in `Like`, we have defined `belongs_to: :photo`, which returns the collection of photos that we're after. We didn't name our new `User` association `:photos` conventionally, so this time we need the `source: :photo`.

Now comes the important `:leaders` association, which defines who a user is following.

The user has `:sent_follow_requests` to people, so we can go `through:` that. And once we get to `sent_follow_requests`, we need to go to the recipients of those follow requests. In `FollowRequest`, we already setup the `belongs_to: :recipient` to return that. So now, we can write:

```ruby{12}
class User < ApplicationRecord
  # ...
  has_many :sent_follow_requests, foreign_key: :sender_id, class_name: "FollowRequest"
  has_many :received_follow_requests, foreign_key: :recipient_id, class_name: "FollowRequest"

  has_many :likes, foreign_key: :fan_id

  has_many :own_photos, foreign_key: :owner_id, class_name: "Photo"

  has_many :liked_photos, through: :likes, source: :photo

  has_many :leaders, through: :sent_follow_requests, source: :recipient
end
```
{: filename="app/models/user.rb" }

But that's not quite all! We need to only return the `:recipient`s who have `"accepted"` in the `FollowRequest` `status` column! That starts out as `"pending"`. Essentially, we want a `.where(status: "accepted")` on the `:sent_follow_requests` before we even get to `:recipient` to filter the requests by.

What we want is _another_ association in `User` called `:accepted_sent_follow_requests`.

This is almost the same as `:sent_follow_requests`, but now with the addition of a second argument to `has_many` that we haven't talked much about: a `Proc` with the syntax `-> {}`. This syntax indicates a small, nameless block of code that will be called on the collection before it is returned. So we can add something like:

```ruby
-> { where(status: "accepted") }
```

This is how we build powerful scoped associations. You can read more about [using scopes in your associations here](https://remimercier.com/scoped-active-record-associations/). We'll return to scopes in a bit when we finish with associations and validations.

Now, finally, we can end up with:

```ruby{4,14}
class User < ApplicationRecord
  # ...
  has_many :sent_follow_requests, foreign_key: :sender_id, class_name: "FollowRequest"
  has_many :accepted_sent_follow_requests, -> { where(status: "accepted") }, foreign_key: :sender_id, class_name: "FollowRequest"

  has_many :received_follow_requests, foreign_key: :recipient_id, class_name: "FollowRequest"

  has_many :likes, foreign_key: :fan_id

  has_many :own_photos, foreign_key: :owner_id, class_name: "Photo"

  has_many :liked_photos, through: :likes, source: :photo

  has_many :leaders, through: :accepted_sent_follow_requests, source: :recipient
end
```
{: filename="app/models/user.rb" }

Did we do that all correctly? It can be hard to tell, but don't worry, the bugs will reveal themselves when we write our sample data, and this is a pretty good start.

While in the `User` model, let's also make an `:accepted_received_follow_requests` association, and the corresponding `:followers` association, with the same logic we just used to get `:leaders`.

We also need a `:pending_received_follow_requests` association and corresponding `:pending` association, so that users with private accounts can see and manage incoming follow requests:

```ruby{7,8,20,22,24}
class User < ApplicationRecord
  # ...
  has_many :sent_follow_requests, foreign_key: :sender_id, class_name: "FollowRequest"
  has_many :accepted_sent_follow_requests, -> { where(status: "accepted") }, foreign_key: :sender_id, class_name: "FollowRequest"

  has_many :received_follow_requests, foreign_key: :recipient_id, class_name: "FollowRequest"
  has_many :accepted_received_follow_requests, -> { where(status: "accepted") }, foreign_key: :recipient_id, class_name: "FollowRequest"
  has_many :pending_received_follow_requests, -> { where(status: "pending") }, foreign_key: :recipient_id, class_name: "FollowRequest"

  has_many :likes, foreign_key: :fan_id

  has_many :own_photos, foreign_key: :owner_id, class_name: "Photo"

  has_many :liked_photos, through: :likes, source: :photo

  has_many :leaders, through: :accepted_sent_follow_requests, source: :recipient

  has_many :followers, through: :accepted_received_follow_requests, source: :sender

  has_many :pending, through: :pending_received_follow_requests, source: :sender

  has_many :feed, through: :leaders, source: :own_photos

  has_many :discover, -> { distinct }, through: :leaders, source: :liked_photos
end
```
{: filename="app/models/user.rb" }

(Note the `-> { distinct }` on the discover association. We need that scope to avoid duplicates when multiple leaders liked the same photo.)

Continuing with the `User` associations accessors (there are quite a few!), we built a `:feed` of photos for each user, which contains the photos _posted_ by their `:leaders`. And a `:discover` page, which contains the photos _liked_ by their `:leaders`.

That's a lot of associations! Isn't it marvelous? You could have used the [association accessor wizard](https://association-accessors.firstdraft.com/), or just think carefully through it.

To write some, or all of these by hand would be really difficult in correct and performant SQL. Rails does it all for us.

Definitely time to `grade`, and, if we see our model tests are passing, git commit and push! (None of the user interface specs are passing yet, but that's okay.)

## Validations

Next, let's write some validations that we think might come in handy for our models. We can go through each column in the table (thanks to our handy `annotaterb` gem) and decide if, and what the validation should be.

We can begin with the `Comment` model:

```ruby{5}
class Comment < ApplicationRecord
  belongs_to :author, class_name: "User", counter_cache: true
  belongs_to :photo, counter_cache: true

  validates :body, presence: true
end
```
{: filename="app/models/comment.rb" }

The only thing we need here is the `presence: true` for the `body` column. We don't really need the comments to be unique or have a certain length.

As we discussed, having a `belongs_to` association will validate the `author_id` and `photo_id` columns by default -- a feature we switched off previously.

For the `FollowRequest` model, we should add some important validations. We want to make sure a user can't send duplicate follow requests, and that users can't follow themselves:

```ruby{6-11}
class FollowRequest < ApplicationRecord
  belongs_to :recipient, class_name: "User"
  belongs_to :sender, class_name: "User"

  enum :status, { pending: "pending", rejected: "rejected", accepted: "accepted" }

  validates :recipient_id, uniqueness: { scope: :sender_id, message: "already requested" }

  validate :users_cant_follow_themselves

  def users_cant_follow_themselves
    if sender_id == recipient_id
      errors.add(:sender_id, "can't follow themselves")
    end
  end
end
```
{: filename="app/models/follow_request.rb" }

The `uniqueness` validation with a `scope` ensures that a user can only have one follow request per recipient. The custom validation `users_cant_follow_themselves` prevents a user from sending a follow request to themselves. (We also added the `enum` here, which we'll discuss in the Enum section below.)

For `Like`, we should also add a uniqueness validation — a user should only be able to like a photo once:

```ruby{5}
class Like < ApplicationRecord
  belongs_to :fan, class_name: "User", counter_cache: true
  belongs_to :photo, counter_cache: true

  validates :fan_id, uniqueness: { scope: :photo_id, message: "has already liked this photo" }
end
```
{: filename="app/models/like.rb" }

On to `Photo`. Aside from foreign keys and columns with default values, only the `caption` and `image` columns need validations:

```ruby{5-6}
class Photo < ApplicationRecord
  # ...
  has_many :fans, through: :likes

  validates :caption, presence: true
  validates :image, presence: true
end
```
{: filename="app/models/photo.rb" }

Finally, we have `User`. All of the password and email things are being taken care of by Devise, so we don't need to worry about them. And we've set defaults on the `_count` columns and `private`.

We should add a validation for the presence and uniqueness of a `username`, along with a format validation to restrict it to letters, numbers, periods, and underscores:

```ruby{5-9}
class User < ApplicationRecord
  # ...
  has_many :discover, -> { distinct }, through: :leaders, source: :liked_photos

  validates :username,
    presence: true,
    uniqueness: true,
    format: {
      with: /\A[\w_\.]+\z/i,
      message: "can only contain letters, numbers, periods, and underscores"
    }
end
```
{: filename="app/models/user.rb" }

When that's done, commit again.

### `strip_attributes` in `ApplicationRecord`

Since we added the `strip_attributes` gem earlier, let's put it to use. By adding `strip_attributes` to `ApplicationRecord`, all models in our app will automatically strip leading and trailing whitespace from their attributes before validation. This prevents issues like users accidentally signing up with `" alice "` as their username:

```ruby
class ApplicationRecord < ActiveRecord::Base
  primary_abstract_class

  strip_attributes
end
```
{: filename="app/models/application_record.rb" }

Commit that change!

## Scopes

Are there any `.where` queries that you know you're going to be using over and over on any of your models? If so, there's a wonderful feature to encapsulate them and make them easy to re-use: [ActiveRecord scopes](https://guides.rubyonrails.org/active_record_querying.html#scopes).

In Photogram, we might frequently want to order photos by newest first, or separate pinned photos from unpinned ones on a user's profile:

```ruby
current_user.own_photos.where(pinned: true)
current_user.own_photos.where(pinned: false).order(created_at: :desc)
```

We can encapsulate these in scopes within our models. For `Photo`:

```ruby{3-5}
class Photo < ApplicationRecord
  # ...
  scope :latest, -> { order(created_at: :desc) }
  scope :pinned, -> { where(pinned: true) }
  scope :unpinned, -> { where(pinned: false) }
end
```
{: filename="app/models/photo.rb" }

We have the `scope` method, the name for the scope (`:latest`, `:pinned`, `:unpinned`), and then a `Proc` defined with the syntax `-> {}`, which just contains our query (`.where`, `.order`) to apply to the model.

And then we can call them on any `ActiveRecord::Relation`s of photos, e.g.:

```ruby
Photo.latest
current_user.own_photos.pinned
current_user.own_photos.unpinned
```

And, if we are careful to write our scopes in such a way that they always return `ActiveRecord::Relation`s, then we can confidently chain them together:

```ruby
current_user.own_photos.unpinned.latest
```

We should also add a scope to the `Comment` model for default ordering (oldest first, so conversations read naturally):

```ruby{5}
class Comment < ApplicationRecord
  belongs_to :author, class_name: "User", counter_cache: true
  belongs_to :photo, counter_cache: true

  scope :default_order, -> { order(created_at: :asc) }

  validates :body, presence: true
end
```
{: filename="app/models/comment.rb" }

Go ahead and add those scopes into the `Photo` and `Comment` models.

Commit that addition!

## `Enum` column type

In `FollowRequest`, we have a column called `status` whose values will be one of only three possibilities: `"pending"`, `"rejected"`, or `"accepted"`. When you find yourself in a situation like this — a column whose possible values are a small fixed list — it's a good candidate to be an `ActiveRecord::Enum`, which will give us a bunch of handy methods for free.

Let's make the column an enum (if you haven't already added it in the Validations section above):

```ruby{5}
class FollowRequest < ApplicationRecord
  belongs_to :recipient, class_name: "User"
  belongs_to :sender, class_name: "User"

  enum :status, { pending: "pending", rejected: "rejected", accepted: "accepted" }
end
```
{: filename="app/models/follow_request.rb" }

We are `enum`erating the list of values (given as a `Hash`) that can be stored in the `status` column. That hash looks redundant, but it's just because people often use integers rather than strings. I prefer strings because they are more meaningful and easy to read.

Now, we automatically get a bunch of handy methods for each status, or each of the keys in our hash. We get `?` and `!` instance methods:

```ruby
# assume follow_request is a valid and pending
follow_request.accepted? # => false
follow_request.accepted! # sets status to "accepted" and saves
```

We also get automatic positive and negative scopes:

```ruby
FollowRequest.accepted
current_user.received_follow_requests.not_rejected
```

Exactly what we need!

In fact, we can now replace the `Proc`s in the association accessors for our `User` model with these handy scopes:

```ruby{4:(49-56),7:(53-60),8:(52-58)}
class User < ApplicationRecord
  # ...
  has_many :sent_follow_requests, foreign_key: :sender_id, class_name: "FollowRequest"
  has_many :accepted_sent_follow_requests, -> { accepted }, foreign_key: :sender_id, class_name: "FollowRequest"

  has_many :received_follow_requests, foreign_key: :recipient_id, class_name: "FollowRequest"
  has_many :accepted_received_follow_requests, -> { accepted }, foreign_key: :recipient_id, class_name: "FollowRequest"
  has_many :pending_received_follow_requests, -> { pending }, foreign_key: :recipient_id, class_name: "FollowRequest"
# ...
```
{: filename="app/models/user.rb" }

And now, because `enum` defines the methods `FollowRequest.accepted` and `FollowRequest.pending` to return filtered lists of follow requests, we can use those methods here!

## Sample data task

At this point, we've done a lot of work! Generated user accounts and a CRUD interface for our domain, added business logic, while also keeping in mind considerations like database indexes and constraints. And we have yet to even start up our web server!

Still, I usually do one more thing before I start working on the interface: write a `sample_data` rake task. It's _so_ helpful to have data to look at when starting to build out functionality and design the interface; and the data should be varied, there should be a significant amount of it, and it should be easy to reset it whenever things get into a weird state while I am experimenting.

Writing a Ruby program to automate this is a huge productivity boost to the whole team, so it's worth doing it up front. The exercise of doing it also invariably helps shake out bugs in the associations and validations that I just wrote.

Since we are using Active Storage for image uploads, the sample data task needs to attach images using Active Storage's `attach` method. Rather than any video content or written guide here, we've supplied you with an example sample data task in the `example_dev.rake` file in the `public/` folder.

Take some time now to read through that file, and copy its contents _exactly_ in your `lib/tasks/dev.rake` file. If you set up your models, associations, and validations correctly, then copying the contents of `public/example_dev.rake` into `lib/tasks/dev.rake` should allow you to create varied sample data by running `rake sample_data` to continue working on the front end user interface in the subsequent lessons.

<aside markdown="1">
The sample data task uses Active Storage's `attach` method to associate images with records. For example, to attach an image to a photo:

```ruby
photo.image.attach(
  io: URI.open(image_url),
  filename: image_url.split("/").last,
  content_type: "image/jpg"
)
```

This is different from the old CarrierWave approach where you'd just set a string column. Active Storage handles the file storage and association behind the scenes.
</aside>

Really, please do spend 30 minutes or so reading through that file carefully and making sure you understand what's going on with sample data creation. And if the sample data task is failing for you, then debug the errors and fix whatever is wrong with your models, associations, and validations.

Running `rake sample_data` at the terminal should now produce very helpful, varied data for our next task: building out the interface! At this point, we're basically done with the whole backend of our app, and now we need to work on the frontend.

- Were you able to copy the `public/example_dev.rake` contents into your `lib/tasks/dev.rake` file and run `rake sample_data` without errors?
- Yes!
  - Great! Carry on!
- Not yet.
  - Please make sure it's working without errors before you proceed!
{: .choose_best #sample_data title="Sample Data" points="1" answer="1" }

- Approximately how long (in minutes) did this lesson take you to complete?
{: .free_text_number #time_taken title="Time taken" points="1" answer="any" }

<!--

# List of project specs for AI assistant

require "rails_helper"

describe "/[USERNAME]/discover" do
  it "can be visited", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/#{user.username}/discover"

    expect(page.status_code).to be(200)
  end

  it "shows photos liked by people the current user follows", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    leader = User.create(username: "leader", email: "leader@example.com", password: "appdev")
    owner = User.create(username: "owner", email: "owner@example.com", password: "appdev", private: false)
    photo = create_photo(owner: owner, caption: "owner caption")
    FollowRequest.create(sender_id: user.id, recipient_id: leader.id, status: "accepted")
    Like.create(fan_id: leader.id, photo_id: photo.id)

    visit "/#{user.username}/discover"

    expect(page).to have_content(photo.caption)
  end
end

def sign_in(user)
  visit "/users/sign_in"

  fill_in "Email", with: user.email
  fill_in "Password", with: user.password
  click_button "Sign in"
end

def create_photo(owner:, caption: "caption")
  photo = Photo.new(caption: caption, owner_id: owner.id)
  photo.image.attach(io: File.open(Rails.root.join("spec/support/test_image.jpeg")), filename: "test_image.jpeg", content_type: "image/jpeg")
  photo.save!
  photo
end

require "rails_helper"

describe "/[USERNAME]/feed" do
  it "can be visited", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/#{user.username}/feed"

    expect(page.status_code).to be(200)
  end

  it "shows their leader's photos", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    leader = User.create(username: "leader", email: "leader@example.com", password: "appdev", private: false)
    photo = create_photo(owner: leader, caption: "leader caption")
    FollowRequest.create(sender_id: user.id, recipient_id: leader.id, status: "accepted")

    visit "/#{user.username}/feed"

    expect(page).to have_content(photo.caption)
    expect(page).to have_css("img")
  end

  it "allows them to like their leader's photos", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    leader = User.create(username: "leader", email: "leader@example.com", password: "appdev", private: false)
    photo = create_photo(owner: leader)
    FollowRequest.create(sender_id: user.id, recipient_id: leader.id, status: "accepted")

    visit "/#{user.username}/feed"

    click_on "0 likes"

    expect(page).to have_css("i.fa-solid.fa-heart")
  end

  it "allows them to un-like their leader's photos", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    leader = User.create(username: "leader", email: "leader@example.com", password: "appdev", private: false)
    photo = create_photo(owner: leader)
    FollowRequest.create(sender_id: user.id, recipient_id: leader.id, status: "accepted")
    Like.create(fan_id: user.id, photo_id: photo.id)

    visit "/#{user.username}/feed"

    click_on "1 like"

    expect(page).to have_css("i.fa-regular.fa-heart")
  end

  it "allows the user to add a comment on their leader's photos", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    leader = User.create(username: "leader", email: "leader@example.com", password: "appdev", private: false)
    photo = create_photo(owner: leader)
    FollowRequest.create(sender_id: user.id, recipient_id: leader.id, status: "accepted")

    visit "/#{user.username}/feed"

    fill_in "comment[body]", with: "New comment"
    click_button "Create Comment"

    expect(page).to have_content("New comment")
  end

  it "allows the user to delete their comment", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    leader = User.create(username: "leader", email: "leader@example.com", password: "appdev", private: false)
    photo = create_photo(owner: leader)
    FollowRequest.create(sender_id: user.id, recipient_id: leader.id, status: "accepted")
    comment = Comment.create(body: "New comment", author_id: user.id, photo_id: photo.id)

    visit "/#{user.username}/feed"

    within("#comment_#{comment.id}") do
      click_on "Delete"
    end

    expect(page).not_to have_content("New comment")
  end

  it "allows the user to edit their comment", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    leader = User.create(username: "leader", email: "leader@example.com", password: "appdev", private: false)
    photo = create_photo(owner: leader)
    FollowRequest.create(sender_id: user.id, recipient_id: leader.id, status: "accepted")
    comment = Comment.create(body: "New comment", author_id: user.id, photo_id: photo.id)

    visit "/#{user.username}/feed"

    within("#comment_#{comment.id}") do
      click_on "Edit"
    end

    fill_in "comment[body]", with: "Edited comment"
    click_button "Update Comment"

    expect(page).to have_content("Edited comment")
  end
end

def sign_in(user)
  visit "/users/sign_in"

  fill_in "Email", with: user.email
  fill_in "Password", with: user.password
  click_button "Sign in"
end

def create_photo(owner:, caption: "caption")
  photo = Photo.new(caption: caption, owner_id: owner.id)
  photo.image.attach(io: File.open(Rails.root.join("spec/support/test_image.jpeg")), filename: "test_image.jpeg", content_type: "image/jpeg")
  photo.save!
  photo
end

require "rails_helper"

describe "/" do
  it "can be visited", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/"

    expect(page.status_code).to be(200)
  end

  it "has a bootstrap navbar", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/"

    expect(page).to have_tag("nav", with: { class: "navbar" })
  end

  it "has a Settings link for the signed in user", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/"

    expect(page).to have_link("Settings", href: "/users/edit")
  end

  it "does not have a sign in link if the user is already signed in", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/"

    expect(page).to_not have_link("Sign in", href: "/users/sign_in")
  end

  it "has a link, 'Feed', that navigates to the 'Feed' page", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/"

    click_on "Feed"

    expect(page).to have_current_path("/#{user.username}/feed")
  end

  it "has a link, 'Discover', that navigates to the 'Discover' page", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/"

    click_on "Discover"

    expect(page).to have_current_path("/#{user.username}/discover")
  end

  it "has a link, 'Go to profile', that navigates to the profile page", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/"

    click_on "Go to profile"

    expect(page).to have_current_path("/#{user.username}")
  end

  it "has an 'Add photo' button", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/"

    expect(page).to have_button("Add photo")
  end
end

def sign_in(user)
  visit "/users/sign_in"

  fill_in "Email", with: user.email
  fill_in "Password", with: user.password
  click_button "Sign in"
end

require "rails_helper"

describe "/photos/new" do
  it "has a form to add a new photo", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/photos/new"

    expect(page).to have_form("/photos", :post)
  end

  it "does not allow the user to add a new photo without a caption", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/photos/new"

    all("input[type='file']").last.attach_file("#{Rails.root}/spec/support/test_image.jpeg")
    all("input[type='submit']").last.click

    expect(page).to have_content("Caption can't be blank")
  end

  it "allows the user to add a new photo", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/photos/new"

    all("input[type='file']").last.attach_file("#{Rails.root}/spec/support/test_image.jpeg")
    all("textarea").last.fill_in(with: "caption")
    all("input[type='submit']").last.click

    expect(page).to have_content("Photo was successfully created")
  end

  it "redirects to the photo details page after creating a new photo", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/photos/new"

    all("input[type='file']").last.attach_file("#{Rails.root}/spec/support/test_image.jpeg")
    all("textarea").last.fill_in(with: "caption")
    all("input[type='submit']").last.click

    expect(page).to have_current_path("/photos/#{Photo.last.id}")
  end
end

describe "/photos/[ID]" do
  it "displays the photo and caption", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    photo = create_photo(owner: user, caption: "caption")

    visit "/photos/#{photo.id}"

    expect(page).to have_css("img")
    expect(page).to have_content(photo.caption)
  end

  it "allows the user to edit the photo", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    photo = create_photo(owner: user, caption: "caption")

    visit "/photos/#{photo.id}"

    click_on "Edit"

    all("textarea").last.fill_in(with: "new caption")
    all("input[type='submit']").last.click

    expect(page).to have_content("new caption")
  end
end

def sign_in(user)
  visit "/users/sign_in"

  fill_in "Email", with: user.email
  fill_in "Password", with: user.password
  click_button "Sign in"
end

def create_photo(owner:, caption: "caption")
  photo = Photo.new(caption: caption, owner_id: owner.id)
  photo.image.attach(io: File.open(Rails.root.join("spec/support/test_image.jpeg")), filename: "test_image.jpeg", content_type: "image/jpeg")
  photo.save!
  photo
end

require "rails_helper"

describe "/[USERNAME]" do
  it "can be visited", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/#{user.username}"

    expect(page.status_code).to be(200)
  end

  it "has a Posts tab", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/#{user.username}"

    expect(page).to have_button("Posts")
  end

  it "has a Likes tab", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/#{user.username}"

    expect(page).to have_button("Likes")
  end

  it "displays each of the user's photos", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    photo = create_photo(owner: user, caption: "caption")

    visit "/#{user.username}"

    expect(page).to have_css("img")
    expect(page).to have_content(photo.caption)
  end

  it "shows the comments on the user's photos", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    photo = create_photo(owner: user, caption: "caption")
    comment = Comment.create(body: "comment body", author_id: user.id, photo_id: photo.id)

    visit "/#{user.username}"

    expect(page).to have_content(comment.body)
  end

  it "allows the user to delete their photo", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    photo = create_photo(owner: user, caption: "caption")

    visit "/#{user.username}"

    click_on "Delete"

    expect(page).not_to have_content(photo.caption)
  end

  it "shows a list of followers on the user profile", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    other_user = User.create(username: "other_user", email: "other_user@example.com", password: "appdev")
    FollowRequest.create(sender_id: other_user.id, recipient_id: user.id, status: "accepted")

    visit "/#{user.username}"

    click_on "followers"

    expect(page).to have_content(other_user.username)
  end

  it "shows a list of leaders on the user profile", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    other_user = User.create(username: "other_user", email: "other_user@example.com", password: "appdev")
    FollowRequest.create(sender_id: user.id, recipient_id: other_user.id, status: "accepted")

    visit "/#{user.username}"

    click_on "following"

    expect(page).to have_content(other_user.username)
  end

  it "shows a 'Following' button for leaders", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    other_user = User.create(username: "other_user", email: "other_user@example.com", password: "appdev")
    FollowRequest.create(sender_id: user.id, recipient_id: other_user.id, status: "accepted")

    visit "/#{other_user.username}"

    expect(page).to have_button("Following")
  end

  it "shows pending follow requests for private accounts", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    private_user = User.create(username: "private_user", email: "private_user@example.com", password: "appdev", private: true)

    visit "/#{private_user.username}"

    click_on "Follow"

    expect(page).to have_button("Requested")
  end

  it "allows a user to unfollow another user", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    other_user = User.create(username: "other_user", email: "other_user@example.com", password: "appdev")
    FollowRequest.create(sender_id: user.id, recipient_id: other_user.id, status: "accepted")

    visit "/#{other_user.username}"

    click_on "Following"

    expect(page).to have_button("Follow")
  end

  it "allows a user to cancel pending follow request", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    private_user = User.create(username: "private_user", email: "private_user@example.com", password: "appdev", private: true)
    FollowRequest.create(sender_id: user.id, recipient_id: private_user.id, status: "pending")

    visit "/#{private_user.username}"

    click_on "Requested"

    expect(page).to have_button("Follow")
  end

  it "allows a user to accept a follow request", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    other_user = User.create(username: "other_user", email: "other_user@example.com", password: "appdev")
    FollowRequest.create(sender_id: other_user.id, recipient_id: user.id, status: "pending")

    visit "/#{user.username}/pending"

    click_on "Accept"

    expect(page).not_to have_content(other_user.username)
  end

  it "allows a user to reject a follow request", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    other_user = User.create(username: "other_user", email: "other_user@example.com", password: "appdev")
    FollowRequest.create(sender_id: other_user.id, recipient_id: user.id, status: "pending")

    visit "/#{user.username}/pending"

    click_on "Reject"

    expect(page).not_to have_content(other_user.username)
  end
end

def sign_in(user)
  visit "/users/sign_in"

  fill_in "Email", with: user.email
  fill_in "Password", with: user.password
  click_button "Sign in"
end

def create_photo(owner:, caption: "caption")
  photo = Photo.new(caption: caption, owner_id: owner.id)
  photo.image.attach(io: File.open(Rails.root.join("spec/support/test_image.jpeg")), filename: "test_image.jpeg", content_type: "image/jpeg")
  photo.save!
  photo
end

require "rails_helper"

describe "User authentication" do
  it "displays a banner to sign in when trying to visit the homepage", points: 1 do
    visit "/"

    expect(page).to have_content("You need to sign in or sign up before continuing")
  end

  it "sends the user to the sign in page when trying to visit the homepage", points: 1 do
    visit "/"

    expect(page).to have_current_path("/users/sign_in")
  end

  it "allows new user sign ups", points: 1 do
    visit "/users/sign_up"

    fill_in "Email", with: "alice@example.com"
    fill_in "Password", with: "appdev"
    fill_in "Password confirmation", with: "appdev"
    fill_in "Username", with: "alice"
    click_button "Sign up"

    expect(page).to have_content("Welcome! You have signed up successfully")
  end

  it "allows an existing user to sign in", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")

    visit "/users/sign_in"

    fill_in "Email", with: user.email
    fill_in "Password", with: user.password
    click_button "Sign in"

    expect(page).to have_content("Signed in successfully")
  end

  it "allows a user to sign out", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")

    visit "/users/sign_in"

    fill_in "Email", with: user.email
    fill_in "Password", with: user.password
    click_button "Sign in"

    click_on user.username
    click_on "Sign out"

    expect(page).to have_current_path("/users/sign_in")
  end
end

require "rails_helper"

RSpec.describe Comment, type: :model do
  describe "has a belongs_to association defined called 'author' with Class name 'User'", points: 1 do
    it { should belong_to(:author).class_name("User") }
  end

  describe "has a belongs_to association defined called 'photo'", points: 1 do
    it { should belong_to(:photo) }
  end
end

require "rails_helper"

RSpec.describe FollowRequest, type: :model do
  describe "has a belongs_to association defined called 'sender' with Class name 'User'", points: 1 do
    it { should belong_to(:sender).class_name("User") }
  end

  describe "has a belongs_to association defined called 'recipient' with Class name 'User'", points: 1 do
    it { should belong_to(:recipient).class_name("User") }
  end
end

require "rails_helper"

RSpec.describe Like, type: :model do
  describe "has a belongs_to association defined called 'fan' with Class name 'User'", points: 1 do
    it { should belong_to(:fan).class_name("User") }
  end
end

RSpec.describe Like, type: :model do
  describe "has a belongs_to association defined called 'photo'", points: 1 do
    it { should belong_to(:photo) }
  end
end

require "rails_helper"

RSpec.describe Photo, type: :model do
  describe "has a belongs_to association defined called 'owner' with Class name 'User'", points: 1 do
    it { should belong_to(:owner).class_name("User") }
  end

  describe "has a has_many association defined called 'comments'", points: 1 do
    it { should have_many(:comments) }
  end

  describe "has a has_many association defined called 'likes'", points: 1 do
    it { should have_many(:likes) }
  end

  describe "has a has_many (many-to_many) association defined called 'fans' through 'likes'", points: 1 do
    it { should have_many(:fans).through(:likes) }
  end
end

require "rails_helper"


RSpec.describe User, type: :model do
  describe "has a has_many association defined called 'comments' with Class name 'Comment' and foreign key 'author_id'", points: 1 do
    it { should have_many(:comments).class_name("Comment").with_foreign_key("author_id") }
  end

  describe "has a has_many association defined called 'own_photos' with Class name 'Photo' and foreign key 'owner_id'", points: 1 do
    it { should have_many(:own_photos).class_name("Photo").with_foreign_key("owner_id") }
  end

  describe "has a has_many association defined called 'likes' with Class name 'Like' and foreign key 'fan_id'", points: 1 do
    it { should have_many(:likes).class_name("Like").with_foreign_key("fan_id") }
  end

  describe "has a has_many (many-to_many) association defined called 'liked_photos' through 'likes' and source 'photo'", points: 1 do
    it { should have_many(:liked_photos).through(:likes).source(:photo) }
  end

  describe "has a has_many association defined called 'sent_follow_requests' with Class name 'FollowRequest' and foreign key 'sender_id'", points: 1 do
    it { should have_many(:sent_follow_requests).class_name("FollowRequest").with_foreign_key("sender_id") }
  end

  describe "has a has_many association defined called 'received_follow_requests' with Class name 'FollowRequest' and foreign key 'recipient_id'", points: 1 do
    it { should have_many(:received_follow_requests).class_name("FollowRequest").with_foreign_key("recipient_id") }
  end

  describe "has a has_many association defined called 'accepted_sent_follow_requests' with scope where 'status' is \"accepted\"", points: 1 do
    it { should have_many(:accepted_sent_follow_requests).class_name("FollowRequest").with_foreign_key("sender_id").conditions(status: "accepted") }
  end

  describe "has a has_many association defined called 'accepted_received_follow_requests' with scope where 'status' is \"accepted\"", points: 1 do
    it { should have_many(:accepted_received_follow_requests).class_name("FollowRequest").with_foreign_key("recipient_id").conditions(status: "accepted") }
  end

  describe "has a has_many (many-to_many) association defined called 'followers' through 'accepted_received_follow_requests' and source 'sender'", points: 1 do
    it { should have_many(:followers).through(:accepted_received_follow_requests).source(:sender) }
  end

  describe "has a has_many (many-to_many) association defined called 'leaders' through 'accepted_sent_follow_requests' and source 'recipient'", points: 1 do
    it { should have_many(:leaders).through(:accepted_sent_follow_requests).source(:recipient) }
  end

  describe "has a has_many (many-to_many) association defined called 'feed' through 'leaders' and source 'own_photos'", points: 1 do
    it { should have_many(:feed).through(:leaders).source(:own_photos) }
  end

  describe "has a has_many (many-to_many) association defined called 'discover' through 'leaders' and source 'liked_photos'", points: 1 do
    it { should have_many(:discover).through(:leaders).source(:liked_photos) }
  end
end

-->
