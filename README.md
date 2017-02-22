# Formex

Formex is an abstract layer that helps to build forms in Phoenix and Ecto. With this library you
don't write changeset, but a separate module that declares fields of form
(like in [Symfony](https://symfony.com/doc/current/forms.html#creating-form-classes)).
Formex will build changeset and additional Ecto queries (to get options for `<select>`) for itself.

Formex also comes with helper functions for templating. For now there is only a Bootstrap 3 form
template, but you can easily create your own templates.

## TL;DR

<img src="http://i.imgur.com/ZwJr3JI.png" width="800px">

## Installation
`mix.exs`
```elixir
def deps do
  [{:formex, "~> 0.3.0"}]
end

def application do
  [applications: [:formex]]
end
```

`config/config.exs`
```elixir
config :formex,
  repo: App.Repo,
  translate_error: &App.ErrorHelpers.translate_error/1,
  template: Formex.Template.BootstrapHorizontal, # optional, can be overridden in a .eex template
  template_options: [ # optional, also can be overridden in the template
    left_column: "col-sm-2",
    right_column: "col-sm-10"
  ]
```

`web/web.ex`
```elixir
def controller do
  quote do
    use Formex.Controller
  end
end

def view do
  quote do
    import Formex.View
  end
end
```

## Usage

We have models Article and Category:

```elixir
schema "articles" do
  field :title, :string
  field :content, :string
  field :hidden, :boolean

  belongs_to :category, App.Category
  many_to_many :tags, App.Tag, join_through: "articles_tags" #...
end
```

```elixir
schema "categories" do
  field :name, :string
end
```

Let's create a form for Article using Formex:
```elixir
# /web/form/article_type.ex
defmodule App.ArticleType do
  use Formex.Type
  alias Formex.CustomField.SelectAssoc

  def build_form(form) do
    form
    |> add(:text_input, :title, label: "Title")
    |> add(:textarea, :content, label: "Content", phoenix_opts: [
      rows: 4
    ])
    |> add(SelectAssoc, :category_id, label: "Category", phoenix_opts: [
      prompt: "Choose a category"
    ])
    |> add(SelectAssoc, :tags, label: "Tags")
    |> add(:checkbox, :hidden, label: "Is hidden?", required: false)
    |> add_button(:submit, "Submit", phoenix_opts: [
      class: "btn-primary"
    ])
  end
end
```

We need to slightly modify a controller:
```elixir
def new(conn, _params) do
  form = create_form(App.ArticleType, %Article{})
  render(conn, "new.html", form: form)
end

def create(conn, %{"article" => article_params}) do
  App.ArticleType
  |> create_form(%Article{}, article_params)
  |> insert_form_data
  |> case do
    {:ok, _article} ->
      conn
      |> put_flash(:info, "Article created successfully.")
      |> redirect(to: article_path(conn, :index))
    {:error, form} ->
      render(conn, "new.html", form: form)
  end
end
```

A template:

`form.html.eex`
```
<%= formex_form_for @form, @action, fn f -> %>
  <%= if @form.changeset.action do %>Error message<% end %>

  <%= formex_row f, :name %>
  <%= formex_row f, :content %>
  <%= formex_row f, :category_id %>
  <%= formex_row f, :submit %>

  <%# or generate all fields at once: formex_rows f %>
<% end %>
```

Also replace `changeset: @changeset` with `form: @form` in `new.html.eex`

Put an asterisk to required fields:
```css
.required .control-label:after {
  content: '*';
  margin-right: 3px;
}
```

The final effect:

<img src="http://i.imgur.com/ojyrWJA.png" width="511px">

It's very simple, isn't it?
You don't need to create any changeset nor write a query to get options for a Category select.
Furthermore, the form code is separated from the template.

## Documentation

[https://hexdocs.pm/formex](https://hexdocs.pm/formex)

### Basic usage
* [Creating forms](https://hexdocs.pm/formex/Formex.Type.html)
* [Usage in a controller](https://hexdocs.pm/formex/Formex.Controller.html)
* [Usage in a template](https://hexdocs.pm/formex/Formex.View.html)

### Custom fields
* [SelectAssoc](https://hexdocs.pm/formex/Formex.CustomField.SelectAssoc.html)
* [Creating a custom field](https://hexdocs.pm/formex/Formex.CustomField.html)

### Templating
* [Changing a template](https://hexdocs.pm/formex/Formex.View.html#module-changing-a-form-template)
* [Creating own template](https://hexdocs.pm/formex/Formex.Template.html)
* [Bootstrap Vertical](https://hexdocs.pm/formex/Formex.Template.BootstrapVertical.html)
* [Bootstrap Horizontal](https://hexdocs.pm/formex/Formex.Template.BootstrapHorizontal.html)

### TODO

- [x] more options for `Formex.CustomField.SelectAssoc`
    - [x] `choice_label`
    - [x] `query`
    - [x] `GROUP BY`
    - [x] multiple_select
- [x] validate if sent `<option>` exists in generated `:select`
- [ ] nested forms
- [x] templating
- [x] tests
- [x] submit button
