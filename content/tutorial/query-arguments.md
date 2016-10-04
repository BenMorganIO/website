---
title: Query Arguments
order: 4
---

Our blog needs users, and the ability to look up users by id. Here's
the query we want to support:

```graphql
# description: Get the name and email address of a user.
{
  user(id: "1") {
    name
    email
  }
}
```

This query includes arguments, which are the key value pairs contained
within the parenthesis. To support this, we'll first create a
user type, and then create a query in our schema that takes an id
argument.


```elixir
# filename: in web/schema/types
@desc "A user of the blog"
object :user do
  field :id, :id
  field :name, :string
  field :email, :string
  field :posts, list_of(:post)
end

@desc "A blog post"
object :post do
  field :title, :string
  field :body, :string
  field :author, :user
end
```

```elixir
# filename: web/schema.ex
query do
  @desc "Get all blog posts"
  field :posts, list_of(:post) do
    resolve &Blog.PostResolver.all/2
  end

  @desc "Get a user of the blog"
  field :user, type: :user do
    arg :id, non_null(:id)
    resolve &Blog.UserResolver.find/2
  end
end
```

In GraphQL you define your arguments ahead of time just like your
return values. This powers a number of very helpful features. To see
them at work, let's look at our resolver.

```elixir
# filename: web/resolvers/user_resolver.ex
defmodule Blog.UserResolver do
  def find(%{id: id}, _info) do
    case Blog.Repo.get(User, id) do
      nil  -> {:error, "User id #{id} not found"}
      user -> {:ok, user}
    end
  end
end
```

Resolve functions are expected to return either `{:ok, item}` or
`{:error, binary | [binary, ...]}`.

The first argument to every resolve function contains the GraphQL
arguments of the query / mutation. Our schema marks the `:id` argument as
`non_null`, so we can be certain we will receive it and just pattern
match directly. If `:id` is left out of the query, Absinthe will
return an informative error to the user, and the resolve function will
not be called.

Note also that the `:id` parameter is an atom, and not a binary like
ordinary phoenix parameters. Absinthe knows what arguments will be
used ahead of time, will coerce as appropriate -- and will cull any extraneous
arguments given to a query. This means that all arguments can be supplied to the
resolve functions with atom keys.

Finally you'll see that we need to handle the possibility that the
query, while valid from GraphQL's perspective, may still ask for a
user that does not exist.
