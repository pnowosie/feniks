![phoenix logo](https://raw.githubusercontent.com/phoenixframework/phoenix/master/priv/static/phoenix.png)
> ### Productive. Reliable. Fast.
> A productive web framework that does not compromise speed and maintainability.

# Who's who
![top contributors](assets/who.png)

# How it works?
[Plug docs](https://hexdocs.pm/plug/readme.html)
```elixir
@doc """
Phoenix is just pipeline of `Plug` modules
Dependencies: it don't even need `cowboy`
"""
def phoenix(%Plug.Conn{} = conn) do
    conn
    |> plug_modules()
end


@doc """
Connection = {Request, Response}
"""
defmodule Plug.Conn do
    defstruct [
        # request fields
        :host,
        :port,
        :path_info,
        :params,

        # response fields
        :status,
        :body,
        :headers
    ]
end


@doc """
Have you heard of `Middlewares`?
"""
defmodule MyPlug do
    def init([]), do: opts
    def call(conn, opts), do: conn
end


@spec plug_fn(Plug.Conn.t()) :: Plug.Conn.t()
```