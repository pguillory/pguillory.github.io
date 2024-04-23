+++
title = 'Postgrex Custom Types'
date = 2024-04-23T13:31:40-07:00
draft = false
+++
## Making UUIDs easier to work with in Postgrex using custom types

I've been using PostgreSQL UUID columns in my Elixir application. It uses the
Postgrex client library to talk to PostgreSQL. It's more typical to use Ecto
(which uses Postgrex under the hood), but I wanted to skip that layer of
abstraction and write raw SQL.

Postgrex returns UUID values as a 16-byte binary.

```elixir
iex> {:ok, conn} = Postgrex.start_link()
iex> Postgrex.query!(conn, "SELECT gen_random_uuid()", []).rows
[[<<83, 100, 109, 250, 55, 215, 75, 173, 188, 226, 250, 2, 111, 128, 117, 42>>]]
```

That's difficult to work with. I'd rather have the string representation, so
that I can drop it in a query parameter or filename without encoding, and so
I can just copy/paste it around.

There's a relatively simple workaround. Postgrex has a framework for
extensions. You can create and install an extension for UUIDs and it will
override the default one. So open up the [Postgrex repo] and find
[uuid.ex]. Copy that into your own repo and edit it. Give the module a new
name, like `MyDatabase.UUID`. In the `encode` function change this:

```elixir
[<<16::int32()>> | uuid]
```

To this:
```elixir
[<<16::int32()>> | UUID.string_to_binary!(uuid)]
```

Then in `decode`, change this:

```elixir
<<16::int32(), uuid::binary-16>> -> uuid
```

To this:

```elixir
<<16::int32(), uuid::binary-16>> -> UUID.binary_to_string!(uuid)
```

Then make a new `.ex` file and put this one line inside:

```elixir
Postgrex.Types.define(MyDatabase.Types, [MyDatabase.UUID])
```

That generates a new module named `MyDatabase.Types`. Now if we use this new
types module when connecting to the database, our UUIDs come back as nicely
encoded strings.

```elixir
iex> {:ok, conn} = Postgrex.start_link(types: MyDatabase.Types)
iex> Postgrex.query!(conn, "SELECT gen_random_uuid()", []).rows
[["4aed7120-bc5b-4e78-995c-79eb46aaac47"]]
```

Note that this requires the `uuid` dependency to be installed in `mix.exs`.

[Postgrex repo]: https://github.com/elixir-ecto/postgrex
[uuid.ex]: https://github.com/elixir-ecto/postgrex/blob/master/lib/postgrex/extensions/uuid.ex
