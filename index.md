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


![](assets/phx-hell.jpg)
# Phoenix meets Blockchain
or API development at imapp(R)

<img src="assets/unhappy-papa.jpg" style="float: right;" height=50% >

## General assumptions
- API follows Http-RPC convention, here action name is contained in url but parameters always go in request body, in JSON
- Accept only POST request with `application/json` set in `Content-type` header
- Responds always with `HTTP 200` for all request successful or not. (`HTTP 500` means erl process crash)
- There is always one additional `application API` layer between Phx controller and application, see: `OMG.Watcher.API` module as example.


## Application API layer
- Does not validates passed parameters, assume valid params or crash early
- Response follows convention of
  - `{:ok, result}` - when successfull
  - `{:error, reason}` - when fails
  - `[list]` - queries with multiple items or empty list


## Controllers layer
- Validates all parameters in `with` expression, use `OMG.RPC.Web.Validator.Base` module
- Pass validated params to `application API` function with the same name as Controller action
- Pass the result to the `api_response` function, (see: `OMG.Watcher.Web` module, which inject this fn to all ctlrs). It does:
  - Unsuccessful result is passed through (and will be handled by `fallback_controller`, it’s std Phx behavior)
  - Sanitizes the result: converts structs to maps, encoded binary to hex, and so (see: `OMG.Watcher.Web.Serializer.Response`)
  - Discovers `View` module from `Ctlr` and passes result with `%{response: result}` where `result` is unwrapped from `{:ok, _}` tuple


## Views layer
- In most cases just passes the response to `OMG.RPC.Web.Serializer.Response.serialize` function which forms std API response 
- It’s the place to restructure response if needed, e.g. add/remove some fields
- It’s assumed that `Views` renders only successful responses
