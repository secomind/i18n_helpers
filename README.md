# I18n Helpers

*I18n Helpers* are a set of tools to help you adding multilingual support to
your Elixir application.

**1. Ease the use of translations stored in database**

   * Translate your Ecto Schema structs (including all Schema associations, in one call)<br>
   ```elixir
   post =
     Repo.all(Post)
     |> Repo.preload(:category)
     |> Repo.preload(:comments)
     |> Translator.translate("fr")

   assert post.translated_title == "Le titre"
   assert post.category.translated_name == "La catégorie"
   assert List.first(post.comments).translated_text == "Un commentaire"
   ```
   * Provide a fallback locale
   ```elixir
   Translator.translate(post, "nl", fallback_locale: "en")
   ```
   * Handle missing translations (e.g. get notified)
   ```elixir
   Translator.translate(post, "en",
     handle_missing_translation: fn translations_map, locale ->
       # add here your error handling stuff,
       # e.g. notify yourself that a translation is missing
     end)
   ```

**2. Fetch the locale from the URL**

   * Assign the locale to the connection and set the Gettext locale
   * Fetch the locale from the request path<br>
     e.g. `example.com/en/hello`, `example.com/fr/bonjour`, …
   * Fetch the locale from the subdomain<br>
     e.g. `en.example.com/hello`, `fr.example.com/bonjour`, …
   * Fetch the locale from the domain name<br>
     e.g. `my-awesome-website.example/hello`, `mon-super-site.example/bonjour`, …
   * Implement a custom locale fetcher

## Translate your Ecto Schema

Translations must be stored in a JSON data type.

> ***Note:** if you prefer to store translations in separate
database tables, then this library (at least the Ecto-related helpers) is not for you. Note however
that, in my opinion, the pros having a JSON field compared to separate translation tables largely
outweigh the cons; but I will not debate that here and let you Google that yourself to form your
own opinion.*

Each translatable field is stored in a map, where each key represents a locale and each value
contains the text for that locale. Below is an example of such map:

```elixir
%{
  "en" => "My Favorite Books",
  "fr" => "Mes Livres Préférés",
  "nl" => "Mijn Lievelingsboeken",
  "en-GB" => "My Favourite Books"
}
```

### What this helper does not

Let's first clarify something important in order to understand what this library actually helps with
and what it does not.

Your translatable text field is essentially a map. In your schema, this translates to:

`field :title, :map`

Inserting/updating/deleting translations is not handled by this library, as nothing specific has to
be done to perform those with `Ecto.Repo` on a `:map` field.

What this library helps with, is **extracting the translations from an Ecto struct and associated
structs into virtual fields based on a given locale**, fallback to a given fallback locale, and
handling missing translations. See examples below.

### Setup your schema

#### without macro

```elixir
defmodule MyApp.Post do
  @behaviour I18nHelpers.Ecto.TranslatableSchema

  use Ecto.Schema

  schema "posts" do
    field :title, :map
    field :translated_title, :string, virtual: true

    field :body, :map
    field :translated_body, :string, virtual: true

    has_many :comments, MyApp.Comment
    belongs_to :category, MyApp.Category
  end

  def get_translatable_fields, do: [:title, :body]
  def get_translatable_assocs, do: [:comments, :category]
end
```

#### with macro

```elixir
defmodule MyApp.Post do
  use Ecto.Schema
  use I18nHelpers.Ecto.TranslatableSchema

  schema "posts" do
    translatable_field :title
    translatable_field :body
    translatable_has_many :comments, MyApp.Comment
    translatable_belongs_to :category, MyApp.Category
  end
end
```

You may import `:i18n_helpers`'s formatter configuration by importing
`i18n_helpers` into your `.formatter.exs` file (this allows for example to keep
`translatable_field :title` without parentheses when running `mix format`).

```elixir
[
  import_deps: [:ecto, :phoenix, :i18n_helpers],
  #...
]
```

The translatable fields in your migration file should also be of `:map` type:

```elixir
add :title, :map, null: false
add :body, :map, null: false
```

### Translate your schema

You will typically translate Schema structs after retrieving them from the database:

```elixir
alias I18nHelpers.Ecto.Translator
alias MyApp.{Post, Repo}

post =
  Repo.all(Post)
  |> Translator.translate("fr")

assert translated_post.translated_title == "Le titre"
assert translated_post.translated_body == "Le contenu"
assert translated_post.category.translated_name == "La catégorie"
```

Note above that all the associated Schema structs have been translated as well.

I prefer to perform translations in the Phoenix controller:

```elixir
Blog.get_post!(post_id) # suppose Blog is the context managing posts, comments, etc.
|> Blog.with_comments_assocs()
|> Blog.with_category_assoc()
|> Translator.translate("fr")
```

Below is an example that shows more clearly the content of the structs and their translations:

```elixir
alias I18nHelpers.Ecto.Translator
alias MyApp.{Category, Comment, Post}

comments = [
  %Comment{text: %{"en" => "A comment", "fr" => "Un commentaire"}},
  %Comment{text: %{"en" => "Another comment", "fr" => "Un autre commentaire"}}
]

category =
  %Category{name: %{"en" => "The category", "fr" => "La catégorie"}}

post =
  %Post{
    title: %{"en" => "The title", "fr" => "Le titre"},
    body: %{"en" => "The content", "fr" => "Le contenu"}
  }
  |> Map.put(:comments, comments)
  |> Map.put(:category, category)

translated_post = Translator.translate(post, "fr")

assert translated_post.translated_title == "Le titre"
assert translated_post.translated_body == "Le contenu"
assert hd(translated_post.comments).translated_text == "Un commentaire"
assert translated_post.category.translated_name == "La catégorie"
```

You can also translate a single field:

```elixir
title = Translator.translate(post.title, "fr") # post.title == %{"en" => "The title", "fr" => "Le titre"}

assert title == "Le titre"
```

#### Locale and fallback locale

If you do not specify the locale to translate to, the library will use the global
[Gettext](https://hexdocs.pm/gettext/Gettext.html) default locale:

```elixir
config :gettext, :default_locale, "fr" # in your `mix.exs` config file

title = Translator.translate(post.title)

assert title == "Le titre"
```

The global locale can be set through a Plug based on the website's host or path (see included plugs
below).

A fallback locale can be given as an option. In the example below, we try to translate the title in
Dutch, but no translation in Dutch has been provided. The translator will then use the given
fallback locale:

```elixir
title = Translator.translate(post.title, "nl", fallback_locale: "en")

assert title == "The title"
```

The default fallback locale is the global Gettext default locale.

#### Handling missing translations

You may provide a callback to handle missing translations:

```elixir
Translator.translate(%{"fr" => "bonjour"}, "en",
  handle_missing_translation: fn translations_map, locale ->

    # add here your error handling stuff,
    # e.g. notify yourself that a translation is missing

    assert translations_map == %{"fr" => "bonjour"}
    assert locale == "en"
  end
)
```

It can be quite tedious to pass your custom callback function to every `translate/3` call; you can
avoid this by wrapping `translate/3` in your own function, where you setup the commonly used
options. You can then import it for every controller through `MyAppWeb.controller/0`. Below is an
example where we want to raise an error when a translation is not found:

```elixir
defmodule MyTranslator do
  alias I18nHelpers.Ecto.Translator

  def translate(data_structure, locale \\ Gettext.get_locale(), opts \\ []) do

    handle_missing_translation =
      Keyword.get(opts, :handle_missing_translation, &handle_missing_translation/2)

    opts =
      Keyword.put(opts, :handle_missing_translation, handle_missing_translation)

    Translator.translate(data_structure, locale, opts)
  end

  def handle_missing_translation(translations_map, locale) do
    raise "missing translation for locale `#{locale}` in #{inspect(translations_map)}"
  end
end
```

## Fetch the locale from the URL

The library provides a set of plugs with different strategies to fetch the locale from the URL.

The plug will assign the locale to the [Connection](https://hexdocs.pm/plug/Plug.Conn.html) and
set the [Gettext](https://hexdocs.pm/gettext/Gettext.html) locale.

You can retrieve the locale from the request path:

```elixir
alias I18nHelpers.Plugs.PutLocaleFromPath

options = PutLocaleFromPath.init(allowed_locales: ["fr", "nl"], default_locale: "en")

conn = conn(:get, "https://example.com/fr/bonjour")
conn = PutLocaleFromPath.call(conn, options)

assert conn.assigns == %{locale: "fr"}
assert Gettext.get_locale == "fr"

conn = conn(:get, "https://example.com/hello") # locale is not specified in path, use `default_locale`
conn = PutLocaleFromPath.call(conn, options)

assert conn.assigns == %{locale: "en"}
assert Gettext.get_locale == "en"
```

or from the subdomain:

```elixir
alias I18nHelpers.Plugs.PutLocaleFromSubdomain

options = PutLocaleFromSubdomain.init(allowed_locales: ["en", "fr"], default_locale: "en")

conn = conn(:get, "https://fr.example.com/bonjour")
conn = PutLocaleFromSubdomain.call(conn, options)

assert conn.assigns == %{locale: "fr"}
assert Gettext.get_locale == "fr"

conn = conn(:get, "https://example.com/hello") # locale is not specified in subdomain, use `default_locale`
conn = PutLocaleFromSubdomain.call(conn, options)

assert conn.assigns == %{locale: "en"}
assert Gettext.get_locale == "en"
```

or from the domain:

```elixir
alias I18nHelpers.Plugs.PutLocaleFromDomain

options =
  PutLocaleFromDomain.init(
    domains_locales_map: %{
      "my-awesome-website.example" => "en",
      "mon-super-site.example" => "fr"
    },
    allowed_locales: ["en", "fr"],
    default_locale: "en"
  )

conn = conn(:get, "https://my-awesome-website.example/hello")
conn = PutLocaleFromDomain.call(conn, options)

assert conn.assigns == %{locale: "en"}
assert Gettext.get_locale == "en"

conn = conn(:get, "https://mon-super-site.example/bonjour")
conn = PutLocaleFromDomain.call(conn, options)

assert conn.assigns == %{locale: "fr"}
assert Gettext.get_locale == "fr"

conn = conn(:get, "https://another-domain.example/hello") # domain not found in `domains_locales_map`, use `default_locale`
conn = PutLocaleFromDomain.call(conn, options)

assert conn.assigns == %{locale: "en"}
assert Gettext.get_locale == "en"
```

or from your custom function: 

```elixir
alias I18nHelpers.Plugs.PutLocale

defp find_locale(conn) do
  case conn.host do
    "en.example.com" ->
      "en"

    "nl.example.com" ->
      "nl"

    _ ->
      case conn.path_info do
        ["en" | _] -> "en"
        ["nl" | _] -> "nl"
        _ -> "en"
      end
  end
end

options = PutLocale.init(find_locale: &find_locale/1)

conn = conn(:get, "/nl/hallo")
conn = PutLocale.call(conn, options)

assert conn.assigns == %{locale: "nl"}
assert Gettext.get_locale == "nl"
```

## Installation

Add `i18n_helpers` for Elixir as a dependency in your `mix.exs` file:

```elixir
def deps do
  [
    {:i18n_helpers, "~> 0.5.3"}
  ]
end
```

## HexDocs

HexDocs documentation can be found at [https://hexdocs.pm/i18n_helpers](https://hexdocs.pm/i18n_helpers).
