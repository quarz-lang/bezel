# bezel

A web framework for [Quarz](https://quarz.dev), in the shape of
Phoenix: an immutable `Conn` flows through plugs, a router and a
controller action, and comes out with a response. Data goes through
[rotor](https://github.com/quarz-lang/rotor) on PostgreSQL
([pg](https://github.com/quarz-lang/pg)). Nothing is looked up by
reflection at run time and no template is interpreted.

```toml
# Cog.toml of an application
[dependencies]
Bezel = { git = "https://github.com/quarz-lang/bezel", branch = "main" }
```

Work in progress (the plan is issue #1). What exists: the connection,
parameters, plugs, endpoints, configuration, error pages, `Bezel.start`,
the built-in plugs, the router and controllers, and the HTML DSL with
forms over rotor's changesets, sessions, the flash, CSRF protection,
secure headers, host authorization, password hashes and signed tokens,
and test helpers. Templates and the `bezel` command come next.

```ruby
class Shop::Endpoint < Bezel::Endpoint
  def self.plugs -> Array[Bezel::Plug]
    [Bezel::Plug::RequestId.new,
     Bezel::Plug::Logger.new,
     Bezel::Plug::Static.new(at: "/", from: "priv/static"),
     Bezel::Plug::Parsers.new(parsers: [:urlencoded, :json], max_body: 1_048_576),
     Bezel::Plug::MethodOverride.new,
     Bezel::Plug::Head.new,
     Bezel::Plug::Timeout.new(15.seconds),
     Bezel.plug { |conn| conn.text("hello #{conn.params.fetch("name", "world")}") }]
  end
end

# HOST, PORT, DATABASE_URL, SECRET_KEY_BASE... from the environment, over
# the application's settings for its BEZEL_ENV (prod when unset: run with
# BEZEL_ENV=dev when developing)
Bezel.start(Shop::Endpoint, {dev: Bezel::Settings.new(database_url: "postgres://localhost/shop_dev"),
                             prod: Bezel::Settings.new(port: 8080)})
```

- **`Bezel::Conn`** is a `data` value: the request, the merged
  parameters (`Bezel::Params`: query, body, path; `params.int("id")`,
  `params["user"].fetch("email")`, a missing or malformed one a `400`),
  assigns, session and flash, and the response being made (status,
  headers, body, `halted?`). Every change makes a new one.
- **A plug** is a `Bezel::Plug` (`call(conn) -> Conn`, and `wrap` for
  one that acts around the rest, as `Logger` and `Timeout` do) or a
  block (`Bezel.plug { |conn| ... }`). A plug that answers halts the
  pipeline; an exception a plug raises halts it too, kept in the
  connection (`conn.error`) with what the plugs before it did.
- **An endpoint** lists its plugs in `def self.plugs`. A request no plug
  answers is a `404`. An exception is answered with the error page:
  `Bezel::NotFound`, `Bezel::Forbidden` and the other `HTTP::Error`s with
  their status, anything else with `500`, reported to the endpoint's
  `on_error`. In `dev` and `test` the page shows the exception, its
  backtrace and the request (HTML, or JSON for a JSON `Accept`); in
  `prod` it shows the status only. Headers the plugs set (the request
  id) stay on the error page.
- **`Bezel::Config`** reads `BEZEL_ENV` (`dev`, `test`, `prod`; `prod` when unset, so that a deploy that forgets it shows no backtraces), `HOST`,
  `PORT`, `DATABASE_URL`, `POOL_SIZE`, `SECRET_KEY_BASE` (required in
  `prod`, 64 bytes or more), `TRUSTED_PROXIES`, `TLS_CERT`/`TLS_KEY` and
  `LOG_LEVEL` over the application's own `Bezel::Settings` per
  environment.
- **`Bezel.start`** serves the endpoint with `HTTP::Server`, opens a
  `PG::Pool` when there is a `DATABASE_URL`, and stops gracefully on
  `SIGTERM` and `SIGINT` (a second signal ends the process at once).

A router declares its routes in one `routes` block, checked where it is
compiled. A controller has one `controller` statement, which generates
the dispatch to its actions and runs its plugs:

```ruby
class Shop::UsersController < Bezel::Controller
  controller do
    plug :require_login, except: [:index, :show]
  end

  def index(conn: Bezel::Conn) -> Bezel::Conn = text(conn, "users")
  def show(conn: Bezel::Conn) -> Bezel::Conn = text(conn, "user #{conn.params.int("id")}")
  # new, create, edit, update, delete ...

  private def require_login(conn: Bezel::Conn) -> Bezel::Conn = ...
end

class Shop::Router < Bezel::Router
  def self.browser -> Array[Bezel::Plug] = [Bezel::Plug::Accepts.new(["html"])]
  def self.api -> Array[Bezel::Plug] = [Bezel::Plug::Accepts.new(["json"])]

  routes do
    scope "/", pipeline: :browser do
      get "/", Shop::PageController, :home, as: "home"
      resources "/users", Shop::UsersController do
        resources "/posts", Shop::PostsController, only: [:index, :show]
      end
    end
    scope "/api", pipeline: :api, as: "api" do
      resources "/users", Shop::API::UsersController, except: [:new, :edit]
    end
  end
end

Shop::Router.user_path(id: 5)                       # => "/users/5"
Shop::Router.user_post_path(user_id: 5, id: 2)      # => "/users/5/posts/2"
```

- A route whose controller lacks its action, a pipeline the router does
  not define, a controller plug that is not a method, and a route helper
  given a wrong keyword are compile errors.
- `resources` makes the seven routes (`index`, `new`, `create`, `show`,
  `edit`, `update` with `PATCH` and `PUT`, `delete`), `only:`/`except:`
  choose, and a block nests routes under `/{user_id}`.
- Matching is the standard `HTTP::Router`'s: the most specific pattern,
  `HEAD` by `GET`, `405` with `Allow` for another method's path.

Views are Quarz code: `Bezel.html` gives a builder with a method per
HTML element, every text and attribute escaped, attributes as typed
keywords (`class_name:` is the `class` attribute, since `class` cannot
name a parameter; `class:` waits for quarz-lang/quarz#382). A layout is
a function taking the inner HTML; a component includes
`Bezel::Component`; a form over a changeset shows its values and errors:

```ruby
module UserView
  def self.edit(conn: Bezel::Conn, cs: Rotor::Changeset[User]) -> Bezel::Html
    Bezel.html(conn) do |h|
      h.h1("Edit #{cs.data.name}")
      h.form_for(cs, action: Shop::Router.user_path(id: cs.data.id), method: :patch) do |f|
        f.label(:email)
        f.email_field(:email)
        f.errors(:email)
        f.submit("Save")
      end
      h.link("Delete", to: Shop::Router.user_path(id: cs.data.id), method: :delete, confirm: "Sure?")
    end
  end
end
```

- A misspelt attribute or one of the wrong type is a compile error;
  `attrs:` passes others (event handlers `on...` are refused), `data:`
  and `aria:` the `data-*` and `aria-*` ones.
- URL attributes, typed or given in `attrs:` (and `srcset`, a meta
  refresh), refuse `javascript:` and other schemes than http, https,
  mailto and tel (relative URLs are fine); `srcdoc` is refused; `<script>`
  and `<style>` refuse content that could end them. A `style` is escaped
  but not checked: do not build one from input.
- Forms carry the CSRF token (from `Bezel.html(conn)`, once the CSRF
  plug sets it) and `_method`, name their fields `user[email]`, and show
  the input as typed for a field with errors. `link`'s `confirm:` writes
  `data-confirm` for the application's script.

A browser pipeline adds sessions and protection:

```ruby
def self.browser -> Array[Bezel::Plug]
  [Bezel::Plug::HostAuthorization.new(["shop.example"]),
   Bezel::Plug::SecureHeaders.new,               # first: error pages get them too
   Bezel::Plug::Session.new(key: "_shop"),      # an encrypted cookie (AES-256-GCM)
   Bezel::Plug::Flash.new,
   Bezel::Plug::CSRF.new]                        # forms from Bezel.html(conn) carry the token
end
```

- The session holds strings (`conn.put_session("user_id", "7")`,
  `conn.session["user_id"]`), encrypted under a key derived from
  `SECRET_KEY_BASE`; its cookie is `HttpOnly`, `SameSite=Lax`, and
  `Secure` outside `dev`. A cookie that does not decrypt is an empty
  session.
- The flash lives one request: `conn.put_flash(:info, "Saved")` before a
  redirect, `conn.flash[:info]` after it.
- CSRF: unsafe methods need the session's token (masked per request) in
  `_csrf_token` or `X-CSRF-Token`, and a request from another origin
  (`Origin`, `Sec-Fetch-Site`) is refused.
- `Bezel::Password.hash/verify` (PBKDF2-SHA256, 600,000 iterations) and
  `Bezel::Token.sign/verify` (HMAC, expiring, bound to a purpose, and to
  a `fingerprint` such as the password hash so that a reset link stops
  working once used) for links sent by mail.
- `conn.session.renew` after a login drops the CSRF secret; `drop` then
  `put` starts a new session.

An application is tested through its endpoint, in the test's task,
without sockets (`Bezel::Testing`):

```ruby
res = Bezel::Testing.get(Shop::Endpoint, "/users")
page = Bezel::Testing.html_response(res, 200)          # a parsed page
page.has_element?("form[action=\"/users\"] input[name=\"user[email]\"]")
page.has_text?("Users")

c = Bezel::Testing::Client.new(Shop::Endpoint, session_key: "_shop")   # keeps cookies
c.login_as({"user_id" => "7"})                          # the session cookie, as a login writes it
c.post("/users", params: {"user[email]" => "ann@example.com"})
c.flash[:info]                                          # => "Welcome", for the next request
c.follow_redirect                                       # which shows it (and takes it)
Bezel::Testing.json_response(c.get("/api/users"), 200)
```

Requests in tests skip the CSRF check (a flag of the test application,
refused in `prod`); `Bezel::Testing.app(skip_csrf: false)` tests it.

## License

MIT No Attribution: see [LICENSE](LICENSE).
