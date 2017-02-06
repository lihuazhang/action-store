ActionStore
-----------

[![Gem Version](https://badge.fury.io/rb/actionstore.svg)](https://badge.fury.io/rb/actionstore) [![Build Status](https://travis-ci.org/rails-engine/actionstore.svg)](https://travis-ci.org/rails-engine/actionstore) [![Code Climate](https://codeclimate.com/github/rails-engine/actionstore/badges/gpa.svg)](https://codeclimate.com/github/rails-engine/actionstore) [![codecov.io](https://codecov.io/github/rails-engine/actionstore/coverage.svg?branch=master)](https://codecov.io/github/rails-engine/actionstore?branch=master) [![](http://inch-ci.org/github/rails-engine/actionstore.svg?branch=master)](http://inch-ci.org/github/rails-engine/actionstore?branch=master)

Store difference kind of actions (Like, Follow, Star, Block ...) in one table via ActiveRecord Polymorphic Association.

- Like Post/Comment/Reply ...
- Watch Post
- Follow User
- Favorite Post

And more and more.

## Basic table struct

| Field | Means |
| ----- | ----- |
| `action_type` | The type of action [like, watch, follow, star, favorite] |
| `action_option` | Secondly option for store you custom status, or you can let it null if you don't needs it. |
| `target_type`, `target_id` | Polymorphic Association for difference `Target` models [User, Post, Comment] |

## Usage

```rb
gem 'actionstore'
```

and run `bundle install`

```bash
rails g action_store:install
```

### Define Actions

You can use `action_for` to define actions:

```
action_for <action_type>, <target>, opts
```

for example:

```rb
# app/models/action.rb
class Action < ActiveRecord::Base
  include ActionStore::Model

  action_for :like, :post, counter_cache: true
  action_for :star, :post, counter_cache: true, user_counter_cache: true
  action_for :follow, :post
  action_for :like, :comment, counter_cache: true
  action_for :follow, :user, counter_cache: 'followers_count', user_counter_cache: 'following_count'
end
```

### Counter Cache

And you need add counter_cache field to target, user table.

```rb
add_column :users, :star_posts_count, :integer, default: 0
add_column :users, :followers_count, :integer, default: 0
add_column :users, :following_count, :integer, default: 0

add_column :posts, :likes_count, :integer, default: 0
add_column :posts, :stars_count, :integer, default: 0

add_column :comments, :likes_count, :integer, default: 0
```

#### Now you can use like this:

@user -> like @post

```rb
irb> Action.create_action(:like, target: @post, user: @user)
irb> @post.reload.likes_count
1
```

@user1 -> unlike @user2

```rb
irb> Action.destroy_action(:follow, target: @post, user: @user)
irb> @post.reload.likes_count
0
```

Check @user1 is liked @post

```rb
irb> action = Action.find_action(:follow, target: @post, user: @user)
irb> action.present?
true
```

User follow cases:

```rb
# @user1 -> follow @user2
Action.create_action(:follow, target: @user2, user: @user1)
@user1.reload.following_count => 1
@user2.reload.followers_count_ => 1
# @user2 -> follow @user1
Action.create_action(:follow, target: @user1, user: @user2)
# @user1 -> follow @user3
Action.create_action(:follow, target: @user3, user: @user1)
# @user1 -> unfollow @user3
Action.destroy_action(:follow, target: @user3, user: @user1)
```

## Builtin relations and methods

When you called `action_for`, ActionStore will define Many-to-Many relations for User and Target model.

for example:

```rb
class Action < ActiveRecord::Base
  include ActionStore::Model

  action_for :like, :post
  action_for :block, :user
end
```

It will defines Many-to-Many relations:

- For User model will defined: `<action>_<target>s` (like_posts)
- For Target model will defined: `<action>_by_users` (like_by_users)

```rb
# for User model
has_many :like_post_actions
has_many :like_posts, through: :like_post_actions
## as user
has_many :block_user_actions
has_many :block_users, through: :block_user_actions
## as target
has_many :block_by_user_actions
has_many :block_by_users, through: :block_by_user_actions

# for Target model
has_many :like_by_user_actions
has_many :like_by_users, through: :like_user_actions
```

And `User` model will have methods:

- @user.like_post(@post)
- @user.unlike_post(@post)
- @user.block_user(@user1)
- @user.unblock_user(@user1)
- @user.like_post_ids
- @user.block_user_ids
- @user.block_by_user_ids