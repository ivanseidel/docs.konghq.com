---
name: Serverless Functions
publisher: Kong Inc.
version: 0.3-x

source_url: https://github.com/Kong/kong-plugin-serverless-functions

desc: Dynamically run Lua code from Kong during the access phase
description: |
  Dynamically run Lua code from Kong during access phase.

type: plugin
categories:
  - serverless

kong_version_compatibility:
    community_edition:
      compatible:
        - 1.2.x
        - 1.1.x
        - 1.0.x
        - 0.14.x
    enterprise_edition:
      compatible:
        - 0.36-x
        - 0.35-x
        - 0.34-x
        - 0.33-x
        - 0.32-x

params:
  name: serverless-functions
  service_id: true
  route_id: true
  consumer_id: false
  protocols: ["http", "https"]
  dbless_compatible: partially
  dbless_explanation: |
    The functions will be executed, but if the configured functions attempt to write to the database, the writes will fail.
  config:
    - name: functions
      required: true
      default: "[]"
      value_in_examples: "[]"
      description: Array of stringified Lua code to be cached and run in sequence during access phase.

---

## Plugin Names

Serverless Functions come as two separate plugins. Each one runs with a
different priority in the plugin chain.

- `pre-function`
  - Runs before other plugins run during access phase.
- `post-function`
  - Runs after other plugins in the access phase.

## Demonstration

### With a Database

1. Create a Service on Kong:

    ```bash
    $ curl -i -X  POST http://localhost:8001/services/ \
      --data "name=plugin-testing" \
      --data "url=http://httpbin.org/headers"

    HTTP/1.1 201 Created
    ...
    ```

2. Add a Route to the Service:

    ```bash
    $ curl -i -X  POST http://localhost:8001/services/plugin-testing/routes \
      --data "paths[]=/test"

    HTTP/1.1 201 Created
    ...
    ```

1. Create a file named `custom-auth.lua` with the following content:

    ```lua
      -- Get list of request headers
      local custom_auth = kong.request.get_header("x-custom-auth")

      -- Terminate request early if our custom authentication header
      -- does not exist
      if not custom_auth then
        return kong.response.exit(401, "Invalid Credentials")
      end

      -- Remove custom authentication header from request
      kong.service.request.clear_header('x-custom-auth')
    ```

4. Ensure the file contents:

    ```bash
    $ cat custom-auth.lua
    ```

5. Apply our Lua code using the `pre-function` plugin using cURL file upload:

    ```bash
    $ curl -i -X POST http://localhost:8001/services/plugin-testing/plugins \
        -F "name=pre-function" \
        -F "config.functions=@custom-auth.lua"

    HTTP/1.1 201 Created
    ...
    ```

6. Test that our Lua code will terminate the request when no header is passed:

    ```bash
    curl -i -X GET http://localhost:8000/test

    HTTP/1.1 401 Unauthorized
    ...
    "Invalid Credentials"
    ```

7. Test the Lua code we just applied by making a valid request:

    ```bash
    curl -i -X GET http://localhost:8000/test \
      --header "x-custom-auth: demo"

    HTTP/1.1 200 OK
    ...
    ```

### Without a Database

1. Create the Service, Route and Associated plugin on the declarative config file:

    ``` yaml
    services:
    - name: plugin-testing
      url: http://httpbin.org/headers

    routes:
    - service: plugin-testing
      paths: [ "/test" ]

    plugins:
    - name: pre-function
      config:
        functions: |
          -- Get list of request headers
          local custom_auth = kong.request.get_header("x-custom-auth")

          -- Terminate request early if our custom authentication header
          -- does not exist
          if not custom_auth then
            return kong.response.exit(401, "Invalid Credentials")
          end

          -- Remove custom authentication header from request
          kong.service.request.clear_header('x-custom-auth')
    ```

2. Test that our Lua code will terminate the request when no header is passed:

    ```bash
    curl -i -X GET http://localhost:8000/test

    HTTP/1.1 401 Unauthorized
    ...
    "Invalid Credentials"
    ```

3. Test the Lua code we just applied by making a valid request:

    ```bash
    curl -i -X GET http://localhost:8000/test \
      --header "x-custom-auth: demo"

    HTTP/1.1 200 OK
    ...
    ```
----

This is just a small demonstration of the power these plugins grant. We were
able to dynamically inject Lua code into the plugin access phase to dynamically
terminate, or transform the request without creating a custom plugin or
reloading / redeploying Kong.

In short, serverless functions give you the full capabilities of a custom plugin
in the access phase without ever redeploying / restarting Kong.


### Notes

#### Upvalues

Prior to version 0.3 of the plugin, the provided Lua code would run as the
function. From version 0.3 onwards also a function can be returned, to allow
for upvalues.

So the older version would do this (still works with 0.3 and above):

```lua
-- this entire block is executed on each request
ngx.log(ngx.ERR, "hello world")
```

With this version version you can return a function to run on each request,
allowing for upvalues to keep state in between requests:

```lua
-- this runs once when Kong starts
local count = 0

return function()
  -- this runs on each request
  count = count + 1
  ngx.log(ngx.ERR, "hello world: ", count)
end
```

#### Minifying Lua

Since we send our code over in a string format, it is advisable to use either
curl file upload `@file.lua` (see demonstration) or to minify your Lua code
using a [minifier][lua-minifier].


[service-url]: https://getkong.org/docs/latest/admin-api/#service-object
[lua-minifier]: https://mothereff.in/lua-minifier
