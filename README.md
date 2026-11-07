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
parameters, plugs, endpoints, configuration, error pages, `Bezel.start`
and the built-in plugs. The router, controllers, views, sessions, test
helpers and the `bezel` command come next.

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

# HOST, PORT, DATABASE_URL, SECRET_KEY_BASE... from the environment,
# over the application's settings per environment
Bezel.start(Shop::Endpoint, {dev: Bezel::Settings.new(database_url: "postgres://localhost/shop_dev")})
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
- **`Bezel::Config`** reads `BEZEL_ENV` (`dev`, `test`, `prod`), `HOST`,
  `PORT`, `DATABASE_URL`, `POOL_SIZE`, `SECRET_KEY_BASE` (required in
  `prod`, 64 bytes or more), `TRUSTED_PROXIES`, `TLS_CERT`/`TLS_KEY` and
  `LOG_LEVEL` over the application's own `Bezel::Settings` per
  environment.
- **`Bezel.start`** serves the endpoint with `HTTP::Server`, opens a
  `PG::Pool` when there is a `DATABASE_URL`, and stops gracefully on
  `SIGTERM` and `SIGINT` (a second signal ends the process at once).

An endpoint is tested without a socket:

```ruby
res = Shop::Endpoint.call(HTTP::Server::Request.get("/?name=Ann"))
res.status     # => 200
```

## License

MIT No Attribution: see [LICENSE](LICENSE).
