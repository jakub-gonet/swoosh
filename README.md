# Swoosh

[![hex.pm](https://img.shields.io/hexpm/v/swoosh.svg)](https://hex.pm/packages/swoosh)
[![hex.pm](https://img.shields.io/hexpm/dt/swoosh.svg)](https://hex.pm/packages/swoosh)
[![hex.pm](https://img.shields.io/hexpm/l/swoosh.svg)](https://hex.pm/packages/swoosh)
[![github.com](https://img.shields.io/github/last-commit/swoosh/swoosh.svg)](https://github.com/swoosh/swoosh)

Compose, deliver and test your emails easily in Elixir.

We have applied the lessons learned from projects like Plug, Ecto and Phoenix
in designing clean and composable APIs, with clear separation of concerns
between modules. Out of the box it comes with 12 adapters, including SendGrid, Mandrill,
Mailgun, Postmark, SMTP... see [Adapters below](#adapters)

The complete documentation for Swoosh is located
[here](https://hexdocs.pm/swoosh).

## Requirements

Elixir 1.9+ and Erlang OTP 22+

## Getting started

```elixir
# In your config/config.exs file
config :sample, Sample.Mailer,
  adapter: Swoosh.Adapters.Sendgrid,
  api_key: "SG.x.x"

# In your application code
defmodule Sample.Mailer do
  use Swoosh.Mailer, otp_app: :sample
end

defmodule Sample.UserEmail do
  import Swoosh.Email

  def welcome(user) do
    new()
    |> to({user.name, user.email})
    |> from({"Dr B Banner", "hulk.smash@example.com"})
    |> subject("Hello, Avengers!")
    |> html_body("<h1>Hello #{user.name}</h1>")
    |> text_body("Hello #{user.name}\n")
  end
end

# In an IEx session
Sample.UserEmail.welcome(%{name: "Tony Stark", email: "tony.stark@example.com"})
|> Sample.Mailer.deliver

# Or in a Phoenix controller
defmodule Sample.UserController do
  use Phoenix.Controller
  alias Sample.UserEmail
  alias Sample.Mailer

  def create(conn, params) do
    user = # create user logic
    UserEmail.welcome(user) |> Mailer.deliver
  end
end
```

See [Mailer docs](https://hexdocs.pm/swoosh/Swoosh.Mailer.html) for more
configuration options.

## Installation

- Add swoosh to your list of dependencies in `mix.exs`:

  ```elixir
  def deps do
    [{:swoosh, "~> 1.0"}]
  end
  ```

- (Optional-ish) Most Adapters (Non SMTP ones) use `Swoosh.ApiClient` to talk
  to the service provider. Swoosh comes with `Swoosh.ApiClient.Hackney`. if you
  want to use the default, include `:hackney` as a dependency as well.
  Otherwise, define a new API client that uses the HTTP client you like, and
  config swoosh to use the new API Client. See `Swoosh.ApiClient` and
  `Swoosh.ApiClient.Hackney` for details.

  ```elixir
  config :swoosh, :api_client, MyApp.ApiClient
  ```

- (Optional) If you are using `Swoosh.Adapters.SMTP`,
  `Swoosh.Adapters.Sendmail` or `Swoosh.Adapters.AmazonSES`, you also need to
  add `gen_smtp` to your deps and list of applications:

  ```elixir
  def deps do
    [
      {:swoosh, "~> 1.0"},
      {:gen_smtp, "~> 0.13"}
    ]
  end
  ```

## Adapters

Swoosh supports the most popular transactional email providers out of the box
and also has a SMTP adapter. Below is the list of the adapters currently
included:

| Provider   | Swoosh adapter                                                                                  |
| ---------- | ----------------------------------------------------------------------------------------------- |
| SMTP       | [Swoosh.Adapters.SMTP](https://hexdocs.pm/swoosh/Swoosh.Adapters.SMTP.html#content)             |
| SendGrid   | [Swoosh.Adapters.Sendgrid](https://hexdocs.pm/swoosh/Swoosh.Adapters.Sendgrid.html#content)     |
| Sendinblue | [Swoosh.Adapters.Sendinblue](https://hexdocs.pm/swoosh/Swoosh.Adapters.Sendinblue.html#content) |
| Sendmail   | [Swoosh.Adapters.Sendmail](https://hexdocs.pm/swoosh/Swoosh.Adapters.Sendmail.html#content)     |
| Mandrill   | [Swoosh.Adapters.Mandrill](https://hexdocs.pm/swoosh/Swoosh.Adapters.Mandrill.html#content)     |
| Mailgun    | [Swoosh.Adapters.Mailgun](https://hexdocs.pm/swoosh/Swoosh.Adapters.Mailgun.html#content)       |
| Mailjet    | [Swoosh.Adapters.Mailjet](https://hexdocs.pm/swoosh/Swoosh.Adapters.Mailjet.html#content)       |
| Postmark   | [Swoosh.Adapters.Postmark](https://hexdocs.pm/swoosh/Swoosh.Adapters.Postmark.html#content)     |
| SparkPost  | [Swoosh.Adapters.SparkPost](https://hexdocs.pm/swoosh/Swoosh.Adapters.SparkPost.html#content)   |
| Amazon SES | [Swoosh.Adapters.AmazonSES](https://hexdocs.pm/swoosh/Swoosh.Adapters.AmazonSES.html#content)   |
| Dyn        | [Swoosh.Adapters.Dyn](https://hexdocs.pm/swoosh/Swoosh.Adapters.Dyn.html#content)               |
| SocketLabs | [Swoosh.Adapters.SocketLabs](https://hexdocs.pm/swoosh/Swoosh.Adapters.SocketLabs.html#content) |
| Gmail      | [Swoosh.Adapters.Gmail](https://hexdocs.pm/swoosh/Swoosh.Adapters.Gmail.html#content)           |

Configure which adapter you want to use by updating your `config/config.exs`
file:

```elixir
config :sample, Sample.Mailer,
  adapter: Swoosh.Adapters.SMTP
  # adapter config (api keys, etc.)
```

Check the documentation of the adapter you want to use for more specific
configurations and instructions.

Adding new adapters is super easy and we are definitely looking for
contributions on that front. Get in touch if you want to help!

## Recipient

The Recipient Protocol enables you to easily make your structs compatible
with Swoosh functions.

```elixir
defmodule MyUser do
  @derive {Swoosh.Email.Recipient, name: :name, address: :email}
  defstruct [:name, :email, :other_props]
end
```

Now you can directly pass `%MyUser{}` to `from`, `to`, `cc`, `bcc`, etc.
See `Swoosh.Email.Recipient` for more details.

## Async Emails

Swoosh does not make any special arrangements for sending emails in a
non-blocking manner.

To send asynchronous emails in Swoosh, one can simply leverage Elixir's
standard library:

```elixir
Task.start(fn ->
  %{name: "Tony Stark", email: "tony.stark@example.com"}
  |> Sample.UserEmail.welcome
  |> Sample.Mailer.deliver
end)
```

Please take a look at the official docs for
[Task](https://hexdocs.pm/elixir/Task.html) and
[Task.Supervisor](https://hexdocs.pm/elixir/Task.Supervisor.html) for further
options.

Note: it is not to say that `Task.start` is enough to cover the whole async
aspect of sending emails. It is more to say that the implementation of sending
emails is very application specific. For example, the simple example above
might be sufficient for some small applications but not so much for more
mission critical applications. Runtime errors, network errors and errors from
the service provider all need to be considered and handled, maybe differently
as well. Whether to retry, how many times you want to retry, what to do when
everything fails, these questions all have different answers in different
context.

## Phoenix integration

If you are looking to use Swoosh in your Phoenix project, make sure to check
out the [phoenix_swoosh](https://github.com/swoosh/phoenix_swoosh) project. It
contains a set of functions that make it easy to render the text and HTML
bodies using Phoenix views, templates and layouts.

Taking the example from above the "Getting Started" section, your code would
look something like this:

```elixir
# web/templates/layout/email.html.eex
<html>
  <head>
    <title><%= @email.subject %></title>
  </head>
  <body>
    <%= @inner_content %>
  </body>
</html>

# web/templates/email/welcome.html.eex
<div>
  <h1>Welcome to Sample, <%= @username %>!</h1>
</div>

# web/emails/user_email.ex
defmodule Sample.UserEmail do
  use Phoenix.Swoosh, view: Sample.EmailView, layout: {Sample.LayoutView, :email}

  def welcome(user) do
    new()
    |> to({user.name, user.email})
    |> from({"Dr B Banner", "hulk.smash@example.com"})
    |> subject("Hello, Avengers!")
    |> render_body("welcome.html", %{username: user.username})
  end
end
```

Feels familiar doesn't it? Head to the
[phoenix_swoosh](https://github.com/swoosh/phoenix_swoosh) repo for more
details.

## Attachments

You can attach files to your email using the `Swoosh.Email.attachment/2`
function. Just give the path of your file as an argument and we will do the
rest. It also works with a `%Plug.Upload{}` struct, or a `%Swoosh.Attachment{}`
struct, which can be constructed using `Swoosh.Attachment.new` detailed here in
the [docs](https://hexdocs.pm/swoosh/Swoosh.Attachment.html#new/2).

All built-in adapters have support for attachments.

```elixir
new()
|> to("peter@example.com")
|> from({"Jarvis", "jarvis@example.com"})
|> subject("Invoice May")
|> text_body("Here is the invoice for your superhero services in May.")
|> attachment("/Users/jarvis/invoice-peter-may.pdf")
```

## Testing

In your `config/test.exs` file set your mailer's adapter to
`Swoosh.Adapters.Test` so that you can use the assertions provided by Swoosh in
`Swoosh.TestAssertions` module.

```elixir
defmodule Sample.UserTest do
  use ExUnit.Case, async: true

  import Swoosh.TestAssertions

  test "send email on user signup" do
    # Assuming `create_user` creates a new user then sends out a `Sample.UserEmail.welcome` email
    user = create_user(%{username: "ironman", email: "tony.stark@example.com"})
    assert_email_sent Sample.UserEmail.welcome(user)
  end
end
```

## Mailbox preview in the browser

Swoosh ships with a Plug that allows you to preview the emails in the local
(in-memory) mailbox. It's particularly convenient in development when you want
to check what your email will look like while testing the various flows of your
application.

For email to reach this mailbox you will need to set your `Mailer` adapter to
`Swoosh.Adapters.Local`:

```elixir
# in config/dev.exs
config :sample, Mailer,
  adapter: Swoosh.Adapters.Local

# to run the preview server alongside your app
# which may not have a web interface already
config :swoosh, serve_mailbox: true

# to change the preview server port (4000 by default)
config :swoosh, serve_mailbox: true, preview_port: 4001
```

In your Phoenix project you can `forward` directly to the plug
without spinning up a separate webserver, like this:

```elixir
# in web/router.ex
if Mix.env == :dev do
  scope "/dev" do
    pipe_through [:browser]

    forward "/mailbox", Plug.Swoosh.MailboxPreview
  end
end
```

And finally you can also use the following Mix task to start the mailbox
preview server independently though note that it won't display/process emails
being sent from outside its own process (great for testing within `iex`).

```console
$ mix swoosh.mailbox.server
```

If you are curious, this is how it looks:

![Plug.Swoosh.MailboxPreview](https://github.com/swoosh/swoosh/raw/main/images/mailbox-preview.png)

The preview is also available as a JSON endpoint.

```sh
$ curl http://localhost:4000/dev/mailbox/json
```

### Production

Swoosh starts a memory storage process for local adapter by default. Normally
it does no harm being left around in production. However, if it is causing
problems, or you don't like having it around, it can be disabled like so:

```elixir
# config/prod.exs
config :swoosh, local: false
```

## Documentation

Documentation is written into the library, you will find it in the source code,
accessible from `iex` and of course, it all gets published to
[hexdocs](http://hexdocs.pm/swoosh).

## Contributing

We are grateful for any contributions. Before you submit an issue or a pull
request, remember to:

- Look at our [Contributing guidelines](CONTRIBUTING.md)
- Not use the issue tracker for help or support requests (try StackOverflow, IRC or Slack instead)
- Do a quick search in the issue tracker to make sure the issues hasn't been reported yet.
- Look and follow the [Code of Conduct](CODE_OF_CONDUCT.md). Be nice and have fun!

### Running tests

Clone the repo and fetch its dependencies:

    $ git clone https://github.com/swoosh/swoosh.git
    $ cd swoosh
    $ mix deps.get
    $ mix test

### Building docs

    $ MIX_ENV=docs mix docs

## LICENSE

See [LICENSE](https://github.com/swoosh/swoosh/blob/main/LICENSE.txt)
