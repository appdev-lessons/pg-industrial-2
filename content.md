# Photogram Industrial: Scaffolds, associations, and validations

## Getting started

Let's continue building our Photogram Industrial project. Here's the target we're working towards:

[pg-industrial.matchthetarget.com](https://pg-industrial.matchthetarget.com/)

Navigate to `github.com/codespaces` (or reopen the previous lesson and use the "Load assignment" button) and reopen your `pg-industrial` project codespace to continue building on what you accomplished in the previous lesson.

At this point, you should have Users and Photos tables with their models configured. In this lesson, we'll generate the remaining three models (Comments, Likes, and FollowRequests), wire up all of the associations between our five models, add validations and scopes, and finally get our sample data running.

Since each lesson builds on the work from the previous one, we want to branch off of our last branch rather than `main`. First, check out the branch from the previous lesson:

```
git checkout create-database
```

Now create a new branch from there:

```
git checkout -b models-and-associations
```

(For a reminder of the new Git workflow we're using, [review this lesson](/lessons/196-git-cli).)

## Generate Comments scaffold

We can generate the rest of our tables with a few more scaffold commands. Let's start with comments (for this and all subsequent scaffold commands use the copy button to avoid typos):

```
rails generate scaffold Comment author:references photo:references body:text
```
{: copyable }

This creates a migration, model, controller, views, and routes for comments. Notice that we used `author:references` instead of `user:references` because we want to call this association `author` because it reads much better in our code (`comment.author` vs `comment.user`).

Let's commit the generated files before we start editing:

```
git add -A
git commit -m "Generated Comments scaffold"
git push --set-upstream origin models-and-associations
```

That last command publishes the branch to GitHub. From now on, you can push with just `git push` after each commit.

[See my commit for this step.](https://github.com/bpurinton/pg-industrial/commit/049f60b79512216617f31c3d4c41c242420168a6)

### Edit the Comments migration

Open the generated migration file in `db/migrate/`. Here's the final version with our edits:

```ruby{4:(55-74),6:(19-31)}
class CreateComments < ActiveRecord::Migration[8.0]
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

[See my commit for this step.](https://github.com/bpurinton/pg-industrial/commit/f5969c9d02d8ed433ee4ed30dd6e8c506b46fc82)

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

[See my commit for this step.](https://github.com/bpurinton/pg-industrial/commit/8390f1154d5e84be6464862d511168a1b270ff97)

### Edit the FollowRequests migration

Open the migration and make these changes:

```ruby{4:(58-77),5:(55-74),6:(23-42)}
class CreateFollowRequests < ActiveRecord::Migration[8.0]
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

```ruby{3}
  # ...
  belongs_to :sender, class_name: "User"

  enum :status, { pending: "pending", rejected: "rejected", accepted: "accepted" }
  # ...
```
{: filename="app/models/follow_request.rb" }

This is one of Rails' most powerful features. An `enum` declaration on a string column does several things automatically:

1. **Query methods**: You can call `follow_request.pending?`, `follow_request.accepted?`, or `follow_request.rejected?` to check the status.
2. **Update methods**: You can call `follow_request.accepted!` to change the status to "accepted" and save the record in one step.
3. **Scopes**: You get `FollowRequest.pending`, `FollowRequest.accepted`, and `FollowRequest.rejected` . These are scopes that return all records with that status.

That last point is particularly important. Later, when we build associations on the User model, we'll use these enum scopes to filter follow requests:

```ruby
has_many :accepted_sent_follow_requests, -> { accepted }, foreign_key: :sender_id, class_name: "FollowRequest"
```

The `-> { accepted }` lambda works because `enum` defined that scope for us. We'll get to this soon.

### Scoped uniqueness validation

Add the scoped uniqueness validation:

```ruby{3}
  # ...
  enum :status, { pending: "pending", rejected: "rejected", accepted: "accepted" }

  validates :recipient_id, uniqueness: { scope: :sender_id, message: "already requested" }
  # ...
```
{: filename="app/models/follow_request.rb" }

This prevents a user from sending multiple follow requests to the same person. The `scope: :sender_id` means "the combination of `recipient_id` and `sender_id` must be unique." Alice can only send one follow request to Bob.

### Custom validation

Add the custom validation method:

```ruby{3,5-9}
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

[See my commit for this step.](https://github.com/bpurinton/pg-industrial/commit/)

## Generate Likes scaffold

The last table we need is Likes — tracking which users have liked which photos:

```
rails generate scaffold Like fan:references photo:references
```
{: copyable }

We're calling the user a `fan` rather than `user` because, again, `like.fan` reads much better than `like.user`.

Commit the generated files:

```
git add -A
git commit -m "Generated Likes scaffold"
git push
```

[See my commit for this step.](https://github.com/bpurinton/pg-industrial/commit/)

### Edit the Likes migration

```ruby{5:(37-68),6:(33-58)}
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

Same pattern as before: `fan` points to the `users` table (so we need `to_table: :users`), while `photo` points to the `photos` table (so `foreign_key: true` works as-is).

### Configure the Like model

```ruby{2-3,5}
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

[See my commit for this step.](https://github.com/bpurinton/pg-industrial/commit/)

## Building out the associations

Now comes the most important part of this lesson. We have five models with basic `belongs_to` associations already in place. But the real power of Rails comes from `has_many` and `has_many :through` associations on the "other side" of each relationship. These let us traverse the social graph with elegant, readable code.

Let's build them up step by step, starting with the Photo model and then the User model.

### Photo model: adding has_many associations

Open `app/models/photo.rb`. Right now it has a `belongs_to :owner` and some validations. Add the `has_many` associations after the `belongs_to`:

```ruby{3,5,7}
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
- `has_many :fans, through: :likes` : this is a **has_many :through** association. It says: "A photo has many fans, _through_ its likes." Rails will follow the chain: Photo -> Likes -> Fan (User). This lets us write `photo.fans` to get all the users who liked a given photo, with no manual joins required.

<div class="alert alert-info">

`has_many :through` is one of the most powerful features in Rails. It lets you traverse a chain of associations to reach a distant related model. The "through" table (in this case, `likes`) acts as a **join table** connecting photos to users. This is how you model many-to-many relationships in Rails.
</div>

### User model: building up step by step

The User model is where all the associations come together. We're going to build it up incrementally. Each version adds a few new lines, highlighted for clarity.

First, let's add the `has_many` for comments. Open `app/models/user.rb`:

```ruby{3}
  # ...
  has_one_attached :profile_banner, dependent: :purge_later

  has_many :comments, foreign_key: :author_id, dependent: :destroy

  has_many :own_photos, foreign_key: :owner_id, class_name: "Photo", dependent: :destroy
  # ...
```
{: filename="app/models/user.rb" }

We need `foreign_key: :author_id` because the `comments` table has a column called `author_id`, not `user_id`. Without specifying the foreign key, Rails would look for `user_id` and find nothing.

Next, let's add the follow request associations, both sent and received:

```ruby{3,5,7,9,11}
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
- `has_many :accepted_sent_follow_requests, -> { accepted }` : a _scoped_ version of sent follow requests that only returns the accepted ones. The `-> { accepted }` lambda uses the scope that our `enum` on FollowRequest created for us. This is where the enum really pays off.
- `has_many :received_follow_requests` : all follow requests _sent to this user_ (where they are the `recipient`).
- `has_many :accepted_received_follow_requests, -> { accepted }` : only the accepted ones among received requests.
- `has_many :pending_received_follow_requests, -> { pending }` : only the pending ones. We'll use this to show a user their pending follow requests that need action.

<aside markdown="1">
Why do we have `dependent: :destroy` on `sent_follow_requests` and `received_follow_requests` but not on the scoped versions (`accepted_sent_follow_requests`, etc.)? Because the scoped versions are just filtered views of the same underlying records. If we put `dependent: :destroy` on those too, Rails would try to destroy the same records multiple times. We only need it on the "base" associations.
</aside>

Now let's add the likes association:

```ruby{3}
  # ...
  has_many :pending_received_follow_requests, -> { pending }, foreign_key: :recipient_id, class_name: "FollowRequest"

  has_many :likes, foreign_key: :fan_id, dependent: :destroy

  has_many :own_photos, foreign_key: :owner_id, class_name: "Photo", dependent: :destroy
  # ...
```
{: filename="app/models/user.rb" }

Again, we need `foreign_key: :fan_id` because the `likes` table uses `fan_id`, not `user_id`.

### The has_many :through associations

Now for the grand finale — the `has_many :through` associations that make our social network actually work. These let us traverse chains of associations to reach distant related models.

Let's walk through each of the new `has_many :through` associations and the other additions.

#### liked_photos

Add the liked_photos association:

```ruby{3}
  # ...
  has_many :own_photos, foreign_key: :owner_id, class_name: "Photo", dependent: :destroy

  has_many :liked_photos, through: :likes, source: :photo
  # ...
```
{: filename="app/models/user.rb" }

"A user has many liked photos, through their likes." Rails follows the chain: User -> Likes -> Photo. The `source: :photo` tells Rails which association on the `Like` model to follow (since `Like` has `belongs_to :photo`). Now we can write `user.liked_photos` to get all the photos a user has liked.

#### leaders and followers

Add the leaders and followers associations:

```ruby{3,5}
  # ...
  has_many :liked_photos, through: :likes, source: :photo

  has_many :leaders, through: :accepted_sent_follow_requests, source: :recipient

  has_many :followers, through: :accepted_received_follow_requests, source: :sender
  # ...
```
{: filename="app/models/user.rb" }

These are the core social graph associations:

- **Leaders**: People I follow (and who accepted my request). Chain: User -> Accepted Sent Follow Requests -> Recipient (User).
- **Followers**: People who follow me (whose requests I accepted). Chain: User -> Accepted Received Follow Requests -> Sender (User).

The `source:` option tells Rails which end of the FollowRequest to grab. For leaders, we want the `:recipient` (the person I sent the request _to_). For followers, we want the `:sender` (the person who sent the request _to me_).

#### pending

Add the pending association:

```ruby{3}
  # ...
  has_many :followers, through: :accepted_received_follow_requests, source: :sender

  has_many :pending, through: :pending_received_follow_requests, source: :sender
  # ...
```
{: filename="app/models/user.rb" }

People who have sent me a follow request that I haven't responded to yet. We'll use this to display a notification count or a list of pending requests.

#### feed — the through-through

Add the feed association:

```ruby{3}
  # ...
  has_many :pending, through: :pending_received_follow_requests, source: :sender

  has_many :feed, through: :leaders, source: :own_photos
  # ...
```
{: filename="app/models/user.rb" }

This is where it gets really exciting. We already have `has_many :leaders` which gives us all the users I follow. Now we go _through_ those leaders to get their `own_photos`. The chain is: User -> Leaders (Users) -> Own Photos (Photos).

In plain English: "My feed is all the photos posted by people I follow." One line of code, and Rails handles the multi-table SQL join for us.

#### discover — the through-through with distinct

Add the discover association:

```ruby{3}
  # ...
  has_many :feed, through: :leaders, source: :own_photos

  has_many :discover, -> { distinct }, through: :leaders, source: :liked_photos

  validates :username,
  # ...
```
{: filename="app/models/user.rb" }

Similar to feed, but instead of our leaders' _own_ photos, we want photos our leaders have _liked_. The chain is: User -> Leaders -> Liked Photos.

The `-> { distinct }` scope is important here. Without it, if two of your leaders both liked the same photo, it would appear twice in your discover feed. The `distinct` ensures each photo appears only once.

<div class="alert alert-info">

Take a moment to appreciate what we just built. With a handful of association declarations, we can now write `user.feed` to get a personalized timeline and `user.discover` to get a recommendation feed. No raw SQL, no complex queries scattered through controllers. Just clean, readable Ruby.
</div>

#### Other additions

There are a few other things we added to the User model beyond the associations:

```ruby{3-4}
  # ...
  validates :website, url: { allow_blank: true }

  attr_accessor :remove_profile_banner
  after_save :purge_profile_banner, if: :remove_profile_banner
  # ...
```
{: filename="app/models/user.rb" }

This virtual attribute and callback let us remove a user's profile banner from a form checkbox. We'll use this in a later lesson when we build the profile edit page.

```ruby{3,5}
  # ...
  after_save :purge_profile_banner, if: :remove_profile_banner

  scope :past_week, -> { where(created_at: 1.week.ago...) }

  scope :by_likes, -> { order(likes_count: :desc) }
  # ...
```
{: filename="app/models/user.rb" }

Two useful scopes: `past_week` returns users who signed up in the last week, and `by_likes` orders users by their like count (most liked first). The `1.week.ago...` syntax is a Ruby beginless range. It means "from one week ago to now."

```ruby{4-6}
  # ...
  scope :by_likes, -> { order(likes_count: :desc) }

  before_create :set_default_avatar

  def self.ransackable_attributes(auth_object = nil)
    [ "username" ]
  end
  # ...
```
{: filename="app/models/user.rb" }

This is required by the `ransack` gem we installed previously. It whitelists which attributes can be searched. We only allow searching by `username`, since we don't want people searching by email or other private fields.

```ruby{3-5}
  # ...
  end

  def purge_profile_banner
    profile_banner.purge_later
  end
end
```
{: filename="app/models/user.rb" }

This is the method called by the `after_save` callback above. It removes the profile banner image from Cloudinary in a background job.

Now would be a good time for a commit:

```
git add -A
git commit -m "Built out all associations, validations, scopes on User and Photo models"
git push
```

[See my commit for this step.](https://github.com/bpurinton/pg-industrial/commit/)

## Run sample data

The starting point includes a pre-written `sample_data` rake task at `lib/tasks/dev.rake`. You don't need to write it; it's already done. Here's what it does at a high level:

- Creates 10 users (Alice through Jack) with emails like `alice@example.com` and the password `appdev`
- Makes some users private (Bob, Carol, Eve, Ivy)
- Attaches specific avatar images from Cloudinary to each user
- Gives Alice a profile banner image
- Creates follow relationships between users (some accepted, some pending)
- Creates 3 photos per user with philosophical captions
- Creates likes and comments from followers
- Uses `User.skip_callback(:create, :before, :set_default_avatar)` to bypass the default avatar callback, since it manually attaches specific avatars for each user

Now that all five models are in place (User, Photo, Comment, Like, and FollowRequest), let's finally try it:

```
rake sample_data
```

This might take a minute or two since it's downloading avatar images from Cloudinary and creating a bunch of records. When it finishes, you should see output indicating that users, photos, follow requests, likes, and comments were all created successfully.

If you get any errors, double-check that:
1. Your `.env` file has valid Cloudinary credentials
2. All five migrations have been run (`rails db:migrate`)
3. Your models match the code shown above

Commit:

```
git add -A
git commit -m "Verified sample data runs successfully"
git push
```

[See my commit for this step.](https://github.com/bpurinton/pg-industrial/commit/)

## Explore the data

Now let's verify that everything is wired up correctly. Start your server with `bin/dev` and visit `/rails/db` in your browser. You should see all of your tables listed. Click into each one and explore:

- **Users**: You should see 10 users (Alice through Jack) with their usernames, emails, and counter columns populated.
- **Photos**: 3 photos per user, 30 total, with captions and `owner_id` values pointing to users.
- **Comments**: Comments on photos with `author_id` and `photo_id` values.
- **Likes**: Records connecting fans to photos.
- **FollowRequests**: A mix of pending and accepted follow requests.

Now let's try the Rails console to see our associations in action:

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

Each of these should return a number (not an error). If `alice.feed` returns photos, that means the full chain of associations is working: User -> Accepted Sent Follow Requests -> Recipients (Leaders) -> Own Photos (Feed). That's a three-table join, expressed as a single method call.

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

1. Generated scaffolds for Comments, FollowRequests, and Likes
2. Edited each migration to use proper foreign keys, constraints, and defaults
3. Configured each model with `belongs_to` associations using `class_name:` when the association name differs from the model name
4. Added `counter_cache: true` to keep count columns in sync automatically
5. Used `enum :status` on FollowRequest to get free query methods and scopes
6. Built `has_many` and `has_many :through` associations on User and Photo
7. Created the `feed` and `discover` through-through associations, the crown jewels of our data model
8. Added validations (both built-in `validates` and custom `validate`)
9. Added scopes for reusable query logic
10. Got `rake sample_data` running and verified everything in the console

The data model is now complete. In the next lessons, we'll turn our attention to the views and controllers, building out the actual pages that users interact with.

Now would be a good time for a final commit and push:

```
git add -A
git commit -m "Completed all models, associations, validations, and sample data"
git push
```

[See my commit for this step.](https://github.com/bpurinton/pg-industrial/commit/)

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
