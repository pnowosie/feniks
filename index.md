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


defmodule Plug.Conn do
@moduledoc """
Connection = {Request, Response}
"""

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


defmodule MyPlug do
@moduledoc """
Have you heard of `Middlewares`?
"""

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
    |> Endpoint.call()
    |> Router.call()
    |> pipeline(:api)
    |> UserController.action()
    |> UserController.index()
    |> UserView.render(:index)
end
```

---

![](assets/phx-hell.jpg)
# Phoenix meets Blockchain
or API development at imapp&reg;

<img src="assets/unhappy-papa.jpg" style="float: left; max-width: 14%; box-sizing: content-box; padding-right: 45px;" >

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

```elixir
defmodule OMG.Watcher.API.Transaction do

    @spec get(binary()) :: {:ok, %DB.Transaction{}} | {:error, :transaction_not_found}
    def get(transaction_id) do
        if transaction = DB.Transaction.get(transaction_id),
            do: {:ok, transaction},
            else: {:error, :transaction_not_found}
    end
end
```


## Controllers layer
- Validates all parameters in `with` expression, use `OMG.RPC.Web.Validator.Base` module
- Pass validated params to `application API` function with the same name as Controller action
- Pass the result to the `api_response` function, (see: `OMG.Watcher.Web` module, which inject this fn to all ctlrs). It does:
  - Unsuccessful result is passed through (and will be handled by `fallback_action`, it’s std Phx behavior)
  - Sanitizes the result: converts structs to maps, encoded binary to hex, 
  - Discovers `View` module from `Ctlr` and passes result with `%{response: result}` where `result` is unwrapped from `{:ok, _}` tuple

```elixir
defmodule OMG.Watcher.Web.Controller.Account do

    def get_balance(conn, params) do
        with {:ok, address} <- expect(params, "address", :address) do
            address
            |> API.Account.get_balance()
            |> api_response(conn, :balance)
        end
    end
end
```


## Views layer
- In most cases just passes the response to `OMG.RPC.Web.Serializer.Response.serialize` function which forms std API response 
- It’s the place to restructure response if needed, e.g. add/remove some fields
- It’s assumed that `Views` renders **only successful responses**

```elixir
defmodule OMG.Watcher.Web.View.Account do

    def render("balance.json", %{response: balance}) do
        balance
        |> OMG.RPC.Web.Response.serialize()
    end
end
```


## Validators
See [validator engine](https://github.com/omisego/elixir-omg/blob/master/apps/omg_rpc/lib/web/validators/base.ex)

* `integer`
* `pos_integer`
* `non_neg_integer`
* `hex` - 0x lowercased, hex-encoded string
* `hash` - :hex + length = 32
* `address` - :hex + length = 20
* `length` - for binary
* `greater` - for integers
* `optional` - :boom: tricky :exclamation:, **always last in validators list**, e.g.

```elixir
  def get_transactions(conn, params) do
    with {:ok, address} <- expect(params, "address", [:address, :optional]),
         {:ok, limit} <- expect(params, "limit", [:pos_integer, :optional]),
         {:ok, blknum} <- expect(params, "blknum", [:pos_integer, :optional]) do

```

In case parameter value fails to satisfy any of validators, tuple `{:error, {:validation_error, param_name, validator}}` is returned and is transalated to client message in `Web.Controller.Fallback`.


## Error handling
In case when castomized error message needs to be provided to client one needs to add unique `atom :error_key` to `@errors` map defined in `Web.Controller.Fallback`. 
Then tuple `{:error, :error_key}` should be returned from corresponding application API's function.


## Resources

* Most recent version of [API Spec in Swagger](https://omisego.github.io/elixir-omg/), 
* Swagger specs `json` is generated from `yaml` files located in `web/priv/swagger` directory, generation is described in [eWallet repo](https://github.com/omisego/ewallet/blob/master/docs/setup/advanced/api_specs.md), 
* [API Errors description page](https://github.com/omisego/elixir-omg/blob/master/docs/api_specs/errors.md)
* [API Spec proposal](https://github.com/omisego/elixir-omg/pull/277)


---
<script type="text/javascript">
    document.querySelector(".markdown-body > :first-child").remove() // remove GH-pages injected header
</script>
