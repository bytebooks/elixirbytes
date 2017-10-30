# Error handling in Plug and Phoenix

### Simple text server

1. Let us build a simple plug based hello world app!
    ```bash
    mix new --sup hello_plug
    ```

1. Add plug and cowboy dependencies
    ```elixir
    # mix.exs

      defp deps do
        [
          {:plug, "> 0.0.0"},
          {:cowboy, "> 0.0.0"},
        ]
      end
    ```

1. Create a plug which can serve a hello world request
    ```elixir
    # lib/hello_plug/web.ex
    defmodule HelloPlug.Web do
      def init(opts), do: opts

      def call(conn, _opts), do: Plug.Conn.send_resp(conn, :ok, "Hello!")
    end
    ```

1. Change Application to start cowboy with our plug on port `3000`
    ```elixir
    # lib/hello_plug/application.ex
    defmodule HelloPlug.Application do
      @moduledoc false

      use Application

      def start(_type, _args) do
        children = [
          Plug.Adapters.Cowboy.child_spec(:http, HelloPlug.Web, [], [port: 3000])
        ]

        opts = [strategy: :one_for_one, name: HelloPlug.Supervisor]
        Supervisor.start_link(children, opts)
      end
    end
    ```
1. Fire up our app
    ```bash
    iex -S mix
    ```
Our sweet little hello page in our browser

![Hello](/images/error_handling_in_plug_and_phoenix/hello_plug_hello.png)

[Check the code for this step on GitHub](https://github.com/bytebooks/error_handling_in_plug_and_phoenix_hello_plug/commit/e2e625f18db583263211c0dbcfe15c70fdf7cb7e)

### The awesome addition machine

Now, let us change our app so that it actually does something useful.

App which can add numbers. Change the plug to the below code:

```elixir
# lib/hello_plug/web.ex
defmodule HelloPlug.Web do
  import Plug.Conn
  def init(opts), do: opts

  def call(conn, _opts) do
    conn = fetch_query_params(conn)

    cond do
      conn.method == "GET" and conn.params == %{} ->
        conn
        |> put_resp_content_type("text/html")
        |> send_resp(:ok, """
                     <!doctype html>
                     <form>
                     <input placeholder="First operand" name="op1" />
                     <input placeholder="Second operand" name="op2" />
                     <button type="submit">Add</button>
                     </form>
                     """)
      conn.method == "GET" ->
        {op1, ""} = conn.params["op1"] |> Integer.parse
        {op2, ""} = conn.params["op2"] |> Integer.parse

        conn
        |> put_resp_content_type("text/html")
        |> send_resp(:ok, """
                     <!doctype html>
                     <a href="/">Go back</a>
                     <p>The sum of #{op1} and #{op2} is #{op1+op2}</p>
                     """)
    end
  end

end
```

**Addition form**  
![Addition form](/images/error_handling_in_plug_and_phoenix/hello_plug_form.png)

**Result of our addition**  
![Result](/images/error_handling_in_plug_and_phoenix/hello_plug_result.png)

[Check the code for this step on GitHub](https://github.com/bytebooks/error_handling_in_plug_and_phoenix_hello_plug/commit/e61267b1e8a0d24141f90157a3fc1b92a7e6ba3d)

### Something crashes!
Having built such a marvelous addition machine we have deployed the code.
However, unbeknownst to us our app is crashing and our users are left frustrated.

One of them is Danny. Danny is trying to do his taxes and has entered 1200.50 and 20.30
![Danny's Taxes](/images/error_handling_in_plug_and_phoenix/hello_plug_taxes_form.png)

However, our addition machine crashes showing a weird page,
Danny is not happy.  
![Danny's Error](/images/error_handling_in_plug_and_phoenix/http_plug_tax_error.png)

Danny ditches our website and looks for another.

We don't want that, do we? So we go and look at the logs one day and see a lot of errors

```
13:48:32.585 [error] #PID<0.284.0> running HelloPlug.Web terminated                            
Server: localhost:3000 (http)                  
Request: GET /?op1=1200.50&op2=20.30           
** (exit) an exception was raised:             
    ** (MatchError) no match of right hand side value: {1200, ".50"}                           
        (hello_plug) lib/hello_plug/web.ex:21: HelloPlug.Web.call/2                            
        (plug) lib/plug/adapters/cowboy/handler.ex:15: Plug.Adapters.Cowboy.Handler.upgrade/4  
        (cowboy) /home/minhajuddin/r/elixirbytes.code/hello_plug/deps/cowboy/src/cowboy_protocol.erl:442: :cowboy_protocol.execute/4
```

It seems our addition machine has crashed because it doesn't understand decimals.

And at this point we want to setup something which shows the user a nice error message and asks the user to enter integers.
We also log the error like a good boy.

```elixir
# lib/hello_plug/web.ex
defmodule HelloPlug.Web do
  # ...

  def call(conn, _opts) do
    try do
      # ...
    rescue
      err ->
        require Logger
        Logger.error("INPUT_ERROR #{inspect err}")
        send_resp(conn, :bad_request, """
                  <!doctype html>
                  <h1 style='color:red;'>Gimme integers!</h1>
                  """)
    end
  end

end
```

[Check the code for this step on GitHub](https://github.com/bytebooks/error_handling_in_plug_and_phoenix_hello_plug/commit/2487070bc04b451eeb5759f0f1f17c46a00094c9)

At this point, our users see a good error message and give us only integers.
However, when we look at the logs there is little information about the error when compared to the previous version.
Just a single line without telling us which line is the culprit.

```
14:02:44.233 [error] INPUT_ERROR %MatchError{term: {1200, ".50"}}
```

So, we decide to just re raise the error and let erlang log it for us.

```elixir
# lib/hello_plug/web.ex
defmodule HelloPlug.Web do
  # ...

  def call(conn, _opts) do
    try do
      # ...
    rescue
      err ->
        send_resp(conn, :bad_request, """
                  <!doctype html>
                  <h1 style='color:red;'>Gimme integers!</h1>
                  """)
        :erlang.raise(:error, err, System.stacktrace())
    end
  end

end
```

And we are back to getting useful error messages in our logs

```
14:09:16.007 [error] #PID<0.291.0> running HelloPlug.Web terminated
Server: localhost:3000 (http)
Request: GET /?op1=1200.50&op2=20.30
** (exit) an exception was raised:
    ** (MatchError) no match of right hand side value: {1200, ".50"}
        (hello_plug) lib/hello_plug/web.ex:22: HelloPlug.Web.call/2
        (plug) lib/plug/adapters/cowboy/handler.ex:15: Plug.Adapters.Cowboy.Handler.upgrade/4
        (cowboy) /home/minhajuddin/r/elixirbytes.code/hello_plug/deps/cowboy/src/cowboy_protocol.erl:442: :cowboy_protocol.execute/4
```
