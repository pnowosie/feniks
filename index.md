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

# The Request pipeline

> Phoenix handles every request in separate Elixir process. Request is translated to `Plug.Conn` structure and goes through pipeline of Plug modules. 

> Plug.call has to return `Plug.Conn` even if it fails to process the request it should return not successful status code.

```elixir
def phoenix(%Plug.Conn{} = conn) do
    conn
    |> endpoint()
    |> router()
    |> pipeline()
    |> controller()
end
```

## The Endpoint
```elixir 
defmodule Hello.Web.Endpoint do
    use Phoenix.Endpoint, otp_app: :hello
    
    plug Plug.Static, ...
    plug Plug.RequestId
    plug Plug.Logger
    
    plug Plug.Parsers, ...
    plug Plug.MethodOverride
    plug Plug.Head
    
    plug Plug.Session, ...
    plug HelloWeb.Router
end
```

## The Router
```elixir 
defmodule Hello.Web.Router do use HelloWeb, :router
    pipeline :browser do
        plug :accepts, ["html"]
        plug :fetch_session
        plug :fetch_flash
        plug :protect_from_forgery
        plug :put_secure_browser_headers
    end

    pipeline :api do
        plug :accepts, ["json", "xml", "gpb"]
    end

    scope "/", HelloWeb do
        pipe_through :browser # Use the default browser stack
        
        get "/", PageController, :index 
    end
end
```

## The Controller
```elixir 
defmodule Hello.Web.UserController do 
    use Hello.Web, :controller

    def index(conn, _params) do
        users = [...]

        conn
        |> put_view(HelloWeb.UserView)
        |> render(:index, users: users)
    end 
end
```

## The View
```elixir
defmodule Hello.Web.UserView do
    use Hello.Web, :view

    def render("index.json", %{users: users}) do
        %{
            version: "1.0",
            data: users
        }
    end
end 
```

## Request pipeline - summary
```elixir
def phoenix(%Plug.Conn{} = conn) do
    conn
    |> endpoint()
    |> router()
    |> pipeline(:api)
    |> UserController.action()
    |> UserController.index()
    |> UserView.render(:index)
end
```

