# Photogram Industrial, Part 2: Scaffolds, Associations, and Validations

## Getting started

Let's continue building our Photogram Industrial project. Here's the target we're working towards:

[pg-industrial.matchthetarget.com](https://pg-industrial.matchthetarget.com/)

Navigate to `github.com/codespaces` (or reopen the previous lesson and use the "Load assignment" button) and reopen your `pg-industrial` project codespace to continue building on what you accomplished in the previous lesson.

At this point, you should have Users and Photos tables with their models configured. In this lesson, we'll generate the remaining three models (Comments, Likes, and FollowRequests), wire up all of the associations between our five models, add validations and scopes, and finally get our sample data running.

Since each lesson builds on the work from the previous one, we want to branch off of our last branch rather than `main`. First, switch to the branch from the previous lesson:

```
git switch create-database
```

Now create a new branch from there:

```
git switch -c models-and-associations
```

For a reminder of the new Git workflow we're using, [review this lesson](/lessons/196-git-cli).

## Generate Comments scaffold

We can generate the rest of our tables with a few more scaffold commands. Let's start with comments (for this and all subsequent scaffold commands use the copy button to avoid typos):

```
rails generate scaffold Comment author:references photo:references body:text
```
{: copyable }

This creates a migration, model, controller, views, and routes for comments. Notice that we used `author:references` instead of `user:references`, because `comment.author` reads better in our code than `comment.user`.

Let's commit the generated files before we start editing:

```
git add -A
git commit -m "Generated Comments scaffold"
git push --set-upstream origin models-and-associations
```

That last command publishes the branch to GitHub. From now on, you can push with just `git push` after each commit.

### Open and submit your pull request

Now that you've pushed your branch to GitHub, it's time to open a **pull request** (PR). A pull request lets us review your code and leave line-by-line feedback.

<div class="alert alert-primary">

<i class="fa-solid fa-circle-play"></i> [Here is a short video demonstration of the process.](https://share.descript.com/view/RLP4apAu5pp) You should also carefully read the notes below!
</div>

On GitHub, navigate to your `pg-industrial` repository. You should see a prompt to open a pull request for the `models-and-associations` branch that you just published, or you can go to the "Pull requests" tab and click "New pull request."

Make sure the **base** branch is `create-database` and the **compare** branch is `models-and-associations`. Also be sure to change the base _repository_ from `appdev-projects/pg-industrial` to _your_ fork. Give it a title, then click "Create pull request."

<aside>

If there's no base _repository_ dropdown when you open your pull request, skip that step. Projects launched from the Grades page now save your work to your own repository rather than a fork, so your pull request already targets your repository.
</aside>

Your PR URL should look like:

```
github.com/[YOUR_GITHUB_USERNAME]/pg-industrial/pull/X
```

It should _not_ contain `appdev-projects` in the URL. If it does, you submitted a PR to _our_ repo instead of to _your own fork_.

<aside>
You don't need to merge your branches now, but when you're ready: [see the notes in our Git CLI lesson](/lessons/196-git-cli#merging-branches).
</aside>

Any additional commits you push to this branch will automatically show up on the pull request. As you work through the rest of this lesson, you can periodically check the PR diff on GitHub in the "Files changed" tab to get a comprehensive overview of everything that has changed so far.

Submit your pull request URL:

- `models-and-associations` compared to `create-database`:
- github.com
  - Great job!
- any
  - Not quite. Make sure the URL looks like: `github.com/[YOUR_GITHUB_USERNAME]/pg-industrial/pull/X`
{: .free_text #pr_url title="Pull request URL" points="1" answer="1" }

### Edit the Comments migration

Open the generated migration file in `db/migrate/`. Here's the final version with our edits:

```ruby{4:(55-74),6:(19-31)}
class CreateComments < ActiveRecord::Migration[8.1]
  def change
    create_table :comments do |t|
      t.references :author, null: false, foreign_key: { to_table: :users }
      t.references :photo, null: false, foreign_key: true
      t.text :body, null: false

      t.timestamps
    end
  end
end
```
{: filename="db/migrate/<date-time-of-migration>_create_comments.rb" }

The key change is on the `author` line. The generator created `foreign_key: true`, which tells the database to look for a table called `authors`. That table doesn't exist. Our table is `users`. So we specify the target table explicitly with `foreign_key: { to_table: :users }`.

We also added `null: false` on the `body` column. A comment without a body shouldn't exist.

### Configure the Comment model

<div class="alert alert-info">

<i class="fa-solid fa-circle-info"></i> Are you still navigating manually through the file tree and clicking to open everything? That's going to become very painful, very quickly. One of the biggest things you can do to increase your productivity is navigating your codebase and its dozens of files without your mouse.

Stop now and experiment with [jumping to files](/lessons/194-helper-methods-part-3#partials-shine-along-with-jump-to-file) in the VSCode fuzzy search bar.
</div>

Open `app/models/comment.rb` and add the following:

```ruby{2:(21-61),3:(20-40),5,7}
class Comment < ApplicationRecord
  belongs_to :author, class_name: "User", counter_cache: true
  belongs_to :photo, counter_cache: true

  validates :body, presence: true

  scope :default_order, -> { order(created_at: :asc) }
end
```
{: filename="app/models/comment.rb" }

Let's walk through this:

- `belongs_to :author, class_name: "User"` : since we named the association `author` instead of `user`, Rails would normally look for an `Author` model. The `class_name: "User"` tells it to use the `User` model instead.
- `counter_cache: true` on both associations : every time a comment is created or destroyed, Rails will automatically update the `comments_count` column on both the associated User _and_ the associated Photo. This avoids expensive `COUNT(*)` queries when displaying "24 comments" on a photo or a user's profile.
- `validates :body, presence: true` : matching our database-level `null: false` constraint with a model-level validation gives the user a friendly error message instead of a database error.
- `scope :default_order` : comments should display oldest-first (ascending), like a conversation.

Now migrate:

```
rails db:migrate
```

And commit:

```
git add -A
git commit -m "Edited Comments migration and configured Comment model"
git push
```

## Generate FollowRequests scaffold

Next up is the `FollowRequest` model. This is the table that powers the social graph. It tracks who wants to follow whom and whether that request has been accepted:

```
rails generate scaffold FollowRequest recipient:references sender:references status
```
{: copyable }

We have two foreign keys here: `recipient` (the person being followed) and `sender` (the person who wants to follow). The `status` column will track whether the request is pending, accepted, or rejected.

Commit the generated files:

```
git add -A
git commit -m "Generated FollowRequests scaffold"
git push
```

### Edit the FollowRequests migration

Open the migration and make these changes:

```ruby{4:(58-77),5:(55-74),6:(23-42)}
class CreateFollowRequests < ActiveRecord::Migration[8.1]
  def change
    create_table :follow_requests do |t|
      t.references :recipient, null: false, foreign_key: { to_table: :users }
      t.references :sender, null: false, foreign_key: { to_table: :users }
      t.string :status, default: "pending"

      t.timestamps
    end
  end
end
```
{: filename="db/migrate/<date-time-of-migration>_create_follow_requests.rb" }

Both `recipient` and `sender` point to the `users` table, so we need `foreign_key: { to_table: :users }` on both. Neither can be null; every follow request must have both a sender and a recipient.

The `status` column has a default of `"pending"`. When a user sends a follow request, it starts as pending until the recipient decides to accept or reject it.

The generator doesn't know about our business logic, so we always need to review and edit generated migrations before running them!

### Configure the FollowRequest model

This model has some interesting features. Let's walk through each addition.

First, update the `belongs_to` associations to specify the User class:

```ruby{2:(24-43),3:(21-40)}
class FollowRequest < ApplicationRecord
  belongs_to :recipient, class_name: "User"
  belongs_to :sender, class_name: "User"
end
```
{: filename="app/models/follow_request.rb" }

Both `recipient` and `sender` point to the `User` model, so we need `class_name: "User"` since the association names don't match the model name. Now let's look at the other additions.

### The enum

Add the enum declaration for the status column:

```ruby{5}
class FollowRequest < ApplicationRecord
  belongs_to :recipient, class_name: "User"
  belongs_to :sender, class_name: "User"

  enum :status, { pending: "pending", rejected: "rejected", accepted: "accepted" }
end
```
{: filename="app/models/follow_request.rb" }

An `enum` declaration on a string column does several things automatically:

1. **Valid values**: The _only_ values the column accepts are `"pending"`, `"rejected"`, or `"accepted"`. Assigning anything else raises an `ArgumentError` (`'foo' is not a valid status`) the moment it's assigned, before any save.
2. **Query methods**: You can call `follow_request.pending?`, `follow_request.accepted?`, or `follow_request.rejected?` to check the status.
3. **Update methods**: You can call `follow_request.accepted!` to change the status to "accepted" and save the record in one step.
4. **Scopes**: You get `FollowRequest.pending`, `FollowRequest.accepted`, and `FollowRequest.rejected`. These are scopes that return all records with that status.
5. **Flexibility**: You can add more values (e.g. `blocked: "blocked"`) to the hash later without changing anything else to get validations, query methods, update methods, and scopes for the new value.

That point about scopes is particularly important. Later, when we build associations on the User model, we'll use these enum scopes to filter follow requests:

```ruby{1:(42-56)}
has_many :accepted_sent_follow_requests, -> { accepted }, foreign_key: :sender_id, class_name: "FollowRequest"
```
{: filename="app/models/user.rb" }

The `-> { accepted }` lambda works because `enum` defined that scope for us:

```ruby{1:(59-78)}
enum :status, { pending: "pending", rejected: "rejected", accepted: "accepted" }
```
{: filename="app/models/follow_request.rb" }

We'll get to this soon.

Let's check which methods an `enum` gives us.

- An `Account` model has `enum :role, { admin: "admin", member: "member" }`. Select all of the methods that Rails defines.
- `Account.find_admin`
  - Not quite. `enum` doesn't define finder methods; the scope is `Account.admin`.
- `Account.admin`
  - Yes! That's the scope for accounts whose role is `"admin"`.
- `account.member?`
  - Yes! That's the query method for one value.
- `account.admin!`
  - Yes! That updates the role to `"admin"` and saves the record.
{: .choose_all #enum_methods title="Methods from an enum" points="1" answer="[2,3,4]" }

### Scoped uniqueness validation

Add the scoped uniqueness validation:

```ruby{4}
  # ...
  enum :status, { pending: "pending", rejected: "rejected", accepted: "accepted" }

  validates :recipient_id, uniqueness: { scope: :sender_id, message: "already requested" }
end
```
{: filename="app/models/follow_request.rb" }

This prevents a user from sending multiple follow requests to the same person. The `scope: :sender_id` means "the combination of `recipient_id` and `sender_id` must be unique." Alice can only send one follow request to Bob.

### Custom validation

Add the custom validation method:

```ruby{4,6-10}
  # ...
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

Notice that this uses `validate` (singular) rather than `validates` (plural). The `validates` method is for built-in validators like `presence`, `uniqueness`, and `format`. The `validate` method is for custom validation methods that you write yourself.

Our custom validation prevents a user from sending a follow request to themselves, which would be nonsensical.

Here's a check on `validate` and `validates`.

- An `Event` model has a method called `ends_after_start`. Which line runs it as a validation?
- `validate :ends_after_start`
  - Yes! `validate` (singular) runs a method we wrote.
- `validates :ends_after_start`
  - Not quite. `validates` (plural) expects an attribute and a built-in validator, like `presence: true`.
- `validates :ends_at, ends_after_start: true`
  - Not quite. That would look for a validator class called `EndsAfterStartValidator`.
{: .choose_best #validate_custom title="Running a custom validation" points="1" answer="1" }

Now migrate:

```
rails db:migrate
```

And commit:

```
git add -A
git commit -m "Edited FollowRequests migration and configured FollowRequest model"
git push
```

## Generate Likes scaffold

The last table we need is Likes, which tracks which users have liked which photos:

```
rails generate scaffold Like fan:references photo:references
```
{: copyable }

We're calling the user a `fan` rather than `user` because, again, `like.fan` reads better than `like.user`.

Commit the generated files:

```
git add -A
git commit -m "Generated Likes scaffold"
git push
```

### Edit the Likes migration

```ruby{4:(52-71)}
class CreateLikes < ActiveRecord::Migration[8.1]
  def change
    create_table :likes do |t|
      t.references :fan, null: false, foreign_key: { to_table: :users }
      t.references :photo, null: false, foreign_key: true

      t.timestamps
    end
  end
end
```
{: filename="db/migrate/<date-time-of-migration>_create_likes.rb" }

Same pattern as before: `fan` points to the `users` table (so we need `to_table: :users`), while `photo` points to the `photos` table (so `foreign_key: true` works as-is).

### Configure the Like model

```ruby{2:(18-58),3:(20-40),5}
class Like < ApplicationRecord
  belongs_to :fan, class_name: "User", counter_cache: true
  belongs_to :photo, counter_cache: true

  validates :fan_id, uniqueness: { scope: :photo_id, message: "has already liked this photo" }
end
```
{: filename="app/models/like.rb" }

The `counter_cache: true` on both associations updates `likes_count` on the User and the Photo whenever a like is created or destroyed. The uniqueness validation prevents a user from liking the same photo twice.

Now migrate:

```
rails db:migrate
```

And commit:

```
git add -A
git commit -m "Edited Likes migration and configured Like model"
git push
```

## Building out the associations

We have five models with basic `belongs_to` associations already in place. Next, we'll add the `has_many` and `has_many :through` associations on the "other side" of each relationship, which let us traverse the social graph starting from any record.

Let's build them up step by step, starting with the Photo model and then the User model.

### Photo model: adding has_many associations

Open `app/models/photo.rb`. Right now it has a `belongs_to :owner` and some validations. Add the `has_many` associations after the `belongs_to`:

```ruby{6,8,10}
# ...
class Photo < ApplicationRecord
  # ...
  belongs_to :owner, class_name: "User", counter_cache: true

  has_many :comments, dependent: :destroy

  has_many :likes, dependent: :destroy

  has_many :fans, through: :likes

  validates :caption, presence: true
  # ...
```
{: filename="app/models/photo.rb" }

The new lines are:

- `has_many :comments, dependent: :destroy` : a photo has many comments. When a photo is deleted, its comments are deleted too.
- `has_many :likes, dependent: :destroy` : same idea for likes.
- `has_many :fans, through: :likes` : this is a **has_many :through** association. It says: "A photo has many fans, _through_ its likes." Rails will follow the chain: Photo → Likes → Fan (User). This lets us write `photo.fans` to get all the users who liked a given photo, with no manual joins required.

<div class="alert alert-info">

<i class="fa-solid fa-circle-info"></i> `has_many :through` lets you traverse a chain of associations to reach a distant related model. The "through" table (in this case, `likes`) acts as a **join table** connecting photos to users. This is how you model many-to-many relationships in Rails.
</div>

### User model: building up step by step

The User model is where all the associations come together. We're going to build it up incrementally. Each version adds a few new lines, highlighted for clarity.

First, let's add the `has_many` for comments. Open `app/models/user.rb`:

```ruby{6}
# ...
class User < ApplicationRecord
  # ...
  has_one_attached :profile_banner, dependent: :purge_later

  has_many :comments, foreign_key: :author_id, dependent: :destroy

  has_many :own_photos, foreign_key: :owner_id, class_name: "Photo", dependent: :destroy
  # ...
```
{: filename="app/models/user.rb" }

We need `foreign_key: :author_id` because the `comments` table has a column called `author_id`, not `user_id`. Without specifying the foreign key, Rails would look for `user_id` and find nothing.

Next, let's add the follow request associations, both sent and received:

```ruby{6,8,10,12,14}
# ...
class User < ApplicationRecord
  # ...
  has_many :comments, foreign_key: :author_id, dependent: :destroy

  has_many :sent_follow_requests, foreign_key: :sender_id, class_name: "FollowRequest", dependent: :destroy

  has_many :accepted_sent_follow_requests, -> { accepted }, foreign_key: :sender_id, class_name: "FollowRequest"

  has_many :received_follow_requests, foreign_key: :recipient_id, class_name: "FollowRequest", dependent: :destroy

  has_many :accepted_received_follow_requests, -> { accepted }, foreign_key: :recipient_id, class_name: "FollowRequest"

  has_many :pending_received_follow_requests, -> { pending }, foreign_key: :recipient_id, class_name: "FollowRequest"

  has_many :own_photos, foreign_key: :owner_id, class_name: "Photo", dependent: :destroy
  # ...
```
{: filename="app/models/user.rb" }

This is a big chunk, so let's unpack it:

- `has_many :sent_follow_requests` : all follow requests _this user sent_ (where they are the `sender`). We need both `foreign_key: :sender_id` and `class_name: "FollowRequest"` because the association name doesn't match the model name.
- `has_many :accepted_sent_follow_requests, -> { accepted }` : a _scoped_ version of sent follow requests that only returns the accepted ones. The `-> { accepted }` lambda uses the scope that our `enum` on FollowRequest created for us.
- `has_many :received_follow_requests` : all follow requests _sent to this user_ (where they are the `recipient`).
- `has_many :accepted_received_follow_requests, -> { accepted }` : only the accepted ones among received requests.
- `has_many :pending_received_follow_requests, -> { pending }` : only the pending ones. We'll use this to show a user their pending follow requests that need action.

<aside>
Why do we have `dependent: :destroy` on `sent_follow_requests` and `received_follow_requests` but not on the scoped versions (`accepted_sent_follow_requests`, etc.)? Because the scoped versions are just filtered views of the same underlying records. If we put `dependent: :destroy` on those too, Rails would try to destroy the same records multiple times. We only need it on the "base" associations.
</aside>

Here's a check on which association does the cleanup.

- When a user is destroyed, which association's `dependent: :destroy` deletes their accepted sent follow requests?
- `accepted_sent_follow_requests`
  - Not quite. That association has no `dependent:` option; it only filters records.
- `sent_follow_requests`
  - Yes! The accepted requests are among the user's sent requests, so the base association removes them.
- `leaders`
  - Not quite. Destroying through `leaders` would delete the other users, not the requests.
{: .choose_best #dependent_destroy title="Which association cleans up" points="1" answer="2" }

Now let's add the likes association:

```ruby{6}
# ...
class User < ApplicationRecord
  # ...
  has_many :pending_received_follow_requests, -> { pending }, foreign_key: :recipient_id, class_name: "FollowRequest"

  has_many :likes, foreign_key: :fan_id, dependent: :destroy

  has_many :own_photos, foreign_key: :owner_id, class_name: "Photo", dependent: :destroy
  # ...
```
{: filename="app/models/user.rb" }

Again, we need `foreign_key: :fan_id` because the `likes` table uses `fan_id`, not `user_id`.

### The has_many :through associations

Next come the `has_many :through` associations on User. These traverse chains of associations to reach distant related models.

Let's walk through each of the new `has_many :through` associations and the other additions.

#### liked_photos

Add the liked_photos association:

```ruby{6}
# ...
class User < ApplicationRecord
  # ...
  has_many :own_photos, foreign_key: :owner_id, class_name: "Photo", dependent: :destroy

  has_many :liked_photos, through: :likes, source: :photo
  # ...
```
{: filename="app/models/user.rb" }

"A user has many liked photos, through their likes." Rails follows the chain: User → Likes → Photo. The `source: :photo` tells Rails which association on the `Like` model to follow (since `Like` has `belongs_to :photo`). Now we can write `user.liked_photos` to get all the photos a user has liked.

#### leaders and followers

Add the leaders and followers associations:

```ruby{6,8}
# ...
class User < ApplicationRecord
  # ...
  has_many :liked_photos, through: :likes, source: :photo

  has_many :leaders, through: :accepted_sent_follow_requests, source: :recipient

  has_many :followers, through: :accepted_received_follow_requests, source: :sender
  # ...
```
{: filename="app/models/user.rb" }

These are the core social graph associations:

- **Leaders**: People I follow (and who accepted my request). Chain: User → Accepted Sent Follow Requests → Recipient (User).
- **Followers**: People who follow me (whose requests I accepted). Chain: User → Accepted Received Follow Requests → Sender (User).

The `source:` option tells Rails which end of the FollowRequest to grab. For leaders, we want the `:recipient` (the person I sent the request _to_). For followers, we want the `:sender` (the person who sent the request _to me_).

Let's check where `source:` points.

- In `has_many :customers, through: :orders, source: :buyer`, where does Rails look for `buyer`?
- A `buyer` column on the `customers` table
  - Not quite. `source:` names an association, not a column.
- A model called `Buyer`
  - Not quite. The records come from whatever class the `buyer` association points to.
- A `belongs_to :buyer` association on `Order`
  - Yes! `source:` names the association to follow on the model we go through.
{: .choose_best #through_source title="Where source points" points="1" answer="3" }

#### pending

Add the pending association:

```ruby{6}
# ...
class User < ApplicationRecord
  # ...
  has_many :followers, through: :accepted_received_follow_requests, source: :sender

  has_many :pending, through: :pending_received_follow_requests, source: :sender
  # ...
```
{: filename="app/models/user.rb" }

People who have sent me a follow request that I haven't responded to yet. We'll use this to display a notification count or a list of pending requests.

#### feed — the through-through

Add the feed association:

```ruby{6}
# ...
class User < ApplicationRecord
  # ...
  has_many :pending, through: :pending_received_follow_requests, source: :sender

  has_many :feed, through: :leaders, source: :own_photos
  # ...
```
{: filename="app/models/user.rb" }

We already have `has_many :leaders` which gives us all the users I follow. Now we go _through_ those leaders to get their `own_photos`. The chain is: User → Leaders (Users) → Own Photos (Photos).

In plain English: "My feed is all the photos posted by people I follow." Rails writes the multi-table SQL join for us.

#### discover — the through-through with distinct

Add the discover association:

```ruby{6}
# ...
class User < ApplicationRecord
  # ...
  has_many :feed, through: :leaders, source: :own_photos

  has_many :discover, -> { distinct }, through: :leaders, source: :liked_photos

  validates :username,
  # ...
```
{: filename="app/models/user.rb" }

Similar to feed, but instead of our leaders' _own_ photos, we want photos our leaders have _liked_. The chain is: User → Leaders → Liked Photos.

The `-> { distinct }` scope is important here. Without it, if two of your leaders both liked the same photo, it would appear twice in your discover feed. The `distinct` ensures each photo appears only once.

### Other additions to the User model

There are a few other things to add to the User model beyond the associations.

#### Removing the profile banner

```ruby{6-7,15-17}
# ...
class User < ApplicationRecord
  # ...
  validates :website, url: { allow_blank: true }

  attr_accessor :remove_profile_banner
  after_save :purge_profile_banner, if: :remove_profile_banner

  before_create :set_default_avatar

  def set_default_avatar
    # ...
  end

  def purge_profile_banner
    profile_banner.purge_later
  end
end
```
{: filename="app/models/user.rb" }

This virtual attribute (`attr_accessor :remove_profile_banner`) and callback (`after_save :purge_profile_banner, if: :remove_profile_banner`) let us remove a user's profile banner from a form checkbox. We also added the `purge_profile_banner` method, which is called by the `after_save` callback. It removes the profile banner image from Cloudinary in a background job. We'll use it in Part 4, when we build the settings form.

#### Two scopes

```ruby{6,7}
# ...
class User < ApplicationRecord
  # ...
  before_create :set_default_avatar

  scope :past_week, -> { where(created_at: 1.week.ago...) }
  scope :by_likes, -> { order(likes_count: :desc) }

  def set_default_avatar
  # ...
```
{: filename="app/models/user.rb" }

Two useful scopes: `past_week` returns users who signed up in the last week, and `by_likes` orders users by their like count (most liked first). The `1.week.ago...` syntax is a Ruby endless range. It means "from one week ago onward."

#### Allowing Ransack searches

```ruby{8-10}
# ...
class User < ApplicationRecord
  # ...
  def purge_profile_banner
    profile_banner.purge_later
  end

  def self.ransackable_attributes(auth_object = nil)
    [ "username" ]
  end
end
```
{: filename="app/models/user.rb" }

This is required by the `ransack` gem we installed previously. It lists the attributes that can be searched. We only allow searching by `username`, since we don't want people searching by email or other private fields.

That was a lot of additions to the User model! Ask any questions now, but we'll also refer back to these methods frequently when we start building things out on the frontend.

Let's make a commit:

```
git add -A
git commit -m "Built out all associations, validations, scopes on User and Photo models"
git push
```

## Run the sample data task

The starting point includes a pre-written `sample_data` rake task at `lib/tasks/dev.rake`. You don't need to write or edit it. Here's what it does:

- Empties the app's five tables and the Active Storage tables.
- Creates 10 users (Alice through Jack) with emails like `alice@example.com` and the password `appdev`, and makes four of them private (Bob, Carol, Eve, and Ivy).
- Attaches an avatar image to each user, and a profile banner to Alice.
- Creates follow requests: Alice, Bob, Carol, and Dave follow everyone else, and Eve, Frank, and Grace send everyone a pending request.
- Creates 3 photos per user, captioned with quotes from Greek philosophers, and pins Alice's first photo.
- Adds likes and comments on each photo from some of its owner's followers.

Most of the file uses the associations we just built. A few snippets are worth a closer look before we run it.

### Making Faker repeatable

```ruby
  # Make Faker output deterministic across runs
  Faker::Config.random = Random.new(42)
```
{: filename="lib/tasks/dev.rake" }

Faker picks its names, hobbies, and quotes at random. Giving it a random number generator with a fixed seed (`42`) makes it pick the same values on every run, so the users' bios and the photos' captions come out the same each time we reset the data.

### Starting from empty tables

```ruby
  ActiveRecord::Base.connection.execute(<<~SQL)
    TRUNCATE TABLE
      follow_requests, comments, likes, photos, users,
      active_storage_variant_records, active_storage_attachments, active_storage_blobs
    RESTART IDENTITY
  SQL
```
{: filename="lib/tasks/dev.rake" }

`TRUNCATE` empties all of the listed tables in one SQL statement. Listing them together lets PostgreSQL empty tables that point at each other with foreign keys. `RESTART IDENTITY` resets each table's ID counter, so after every run the first user's ID is `1` again.

### Attaching images without uploading them

```ruby
    # Attach before saving, so the default-avatar callback sees an avatar and
    # skips itself
    attach_stable_cloudinary_url(user.avatar_image, avatar_images[counter])
```
{: filename="lib/tasks/dev.rake" }

The `attach_stable_cloudinary_url` method is defined at the bottom of the file. It doesn't upload anything. The sample images are already hosted on our First Draft Cloudinary account, so the method creates an Active Storage blob record that points at one of them, and attaches that blob. Running the task doesn't add any files to your Cloudinary account.

The avatar is attached before `user.save!`, and the comment says that the default-avatar callback then skips itself. Ours doesn't yet: `before_create :set_default_avatar` runs for every new user, downloads the default image, and replaces the avatar the task attached. Add a condition to the callback in the User model:

```ruby{4:(36-74)}
  # ...
  after_save :purge_profile_banner, if: :remove_profile_banner

  before_create :set_default_avatar, unless: -> { avatar_image.attached? }
  # ...
```
{: filename="app/models/user.rb" }

`unless:` takes a lambda that Rails runs just before the callback. When the lambda returns `true`, the callback is skipped. Users who sign up through the app don't have an avatar attached yet, so they still get the default one.

### Building the follow graph

```ruby
      if first_user.username.in?(%w[alice bob carol dave])
        first_user.sent_follow_requests.create(
          recipient: second_user,
          status: "accepted"
        )
      end
```
{: filename="lib/tasks/dev.rake" }

Creating the request through `first_user.sent_follow_requests` fills in `sender_id` with `first_user`'s ID for us, so we only pass the `recipient` and the `status`.

### Liking a photo through an association

```ruby
        if follower.username.in?(%w[alice bob eve frank])
          photo.fans << follower
        end
```
{: filename="lib/tasks/dev.rake" }

`photo.fans << follower` writes through the `has_many :fans, through: :likes` association. Rails creates the `Like` record that connects the photo and the follower, and the counter caches on both of them go up by one.

### Run it

Now that all five models are in place (User, Photo, Comment, Like, and FollowRequest), let's try it:

```
rake sample_data
```

It takes a little while. When it finishes, the last lines of output should be:

```
"There are now 10 users."
"There are now 63 follow requests."
"There are now 30 photos."
"There are now 54 likes."
"There are now 54 comments."
```

If you get any errors, double-check that:

1. Your `.env` file has valid Cloudinary credentials.
2. All five migrations have been run (`rails db:migrate`).
3. Your models match the code in this lesson, including the `unless:` condition on `set_default_avatar`.

If anything is missing, or you hit a bug, try to resolve things and commit and push before you move on from here.

<div class="alert alert-success">

<i class="fa-solid fa-circle-check"></i> Run `rake sample_data` a second time. The counts come out the same, because the task empties the tables before it creates anything.
</div>

Here's a check on the task.

- After you run `rake sample_data` again, what does `User.first.id` return in `rails console`?
- 1
  - Yes! `RESTART IDENTITY` starts the IDs at 1 again on every run.
- any
  - Not quite. Try it in `rails console`.
{: .free_text_number #first_user_id title="The first user's ID" points="1" answer="1" }

<div class="alert alert-info">

<i class="fa-solid fa-circle-info"></i> The sample data task is a _very_ important step after finishing your database work. Writing, running, and exploring sample data tasks often shakes out bugs or deficiencies in the data model that we want to resolve _before_ we build a whole frontend on top of it.

In the future _you_ will need to write your own sample data task, and you should do it very early before writing all of your front end code.
</div>

## Explore the data

Now let's verify that everything is wired up correctly, using the Rails console to see our associations in action:

```
rails console
```

Try these commands one at a time:

```ruby
alice = User.find_by(username: "alice")
alice.own_photos.count
alice.liked_photos.count
alice.leaders.count
alice.followers.count
alice.feed.count
alice.discover.count
alice.pending.count
```

Each of these should return a number (not an error). With the sample data, Alice's numbers are 3, 27, 9, 3, 27, 27, and 3. If `alice.feed` returns photos, that means the full chain of associations is working: User → Accepted Sent Follow Requests → Recipients (Leaders) → Own Photos (Feed).

You can also test the other direction:

```ruby
photo = Photo.first
photo.owner.username
photo.fans.count
photo.comments.count
```

And check a follow request:

```ruby
fr = FollowRequest.first
fr.sender.username
fr.recipient.username
fr.status
fr.accepted?
```

That `.accepted?` method came for free from our `enum` declaration.

Exit the console with `exit` when you're done exploring.

## Recap

In this lesson, we:

1. Generated scaffolds for Comments, FollowRequests, and Likes.
2. Edited each migration to use proper foreign keys, constraints, and defaults.
3. Configured each model with `belongs_to` associations using `class_name:` when the association name differs from the model name.
4. Added `counter_cache: true` to keep count columns in sync automatically.
5. Used `enum :status` on FollowRequest to get free query methods and scopes.
6. Built `has_many` and `has_many :through` associations on User and Photo.
7. Created the `feed` and `discover` associations, which each go through two other associations.
8. Added validations (both built-in `validates` and custom `validate`).
9. Added scopes for reusable query logic.
10. Added a condition to the default-avatar callback, got `rake sample_data` running, and verified everything in the console.

The data model is now complete. In the next lessons, we'll turn our attention to the views and controllers, building out the actual pages that users interact with.

If anything is missing, or you hit a bug, try to resolve things and commit and push before you move on from here.

Now that all five models exist, the tests can load.

<div class="alert alert-success">

<i class="fa-solid fa-circle-check"></i> Run `rspec`. You should see `66 examples, 44 failures`: the 22 model tests pass, and the 44 tests that visit pages fail until we build those pages in Parts 3 and 4. Then run `grade`, so that Grades records the passing tests.
</div>

---

- Approximately how long (in minutes) did this lesson take you to complete?
{: .free_text_number #time_taken title="Time taken" points="1" answer="any" }

<!--

# List of project specs for AI assistant

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

-->
