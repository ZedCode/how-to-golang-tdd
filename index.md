# Using Test Driven Development in Go to bind services together

One of the most common problems I've run into in the enterprise environment is binding some ancient service to an API for easy consumption by their fancy new React SPA as they attempt to lift and shift old applications from a traditional cludge of scripts and potentially monolithic installation into a nice decomposed application.

In this post I'll walk you through how I have tackled this problem dozens of times over the last few years using an example that would be extremely common.

## The Problem

In this hypothetical scenario we're going to build a tool which can scan VLANs by shelling out and using `nmap` from within a blessed host. In most environments, just running an `nmap` scan against a bunch of hosts to learn things like "did all my VMs become response after deployment?" is hopefully going to set off an IDS. In this example we have a blessed host which nobody will care about nmap traffic coming from, we need to write well tested code which will give SecOps the confidence to let us deploy this into our environment to help increase our automation's visibility :)

## A Word About Test Driven Development

In the book "Cloud Native Go" it is emphasized that in order for your tests to drive your development, you must first write a broken test and then get your test to pass by writing the code required to complete the test successfully. More importantly though is what ends up happening when you do this: you write *only the code you need -- nothing more.* As I walk through how I'd tackle this problem space, consider how if we were not writing tests we might accidentally introduce a lot of logic which is beyond the scope of our requirements. In addition to introducing incomplete code which might be in various states of tested, this increases the attack surface of your API for no reason. While the strictest interpretations of TDD are not something I'm going to walk through here, we're primarily embracing any philosophy that minimizes extra code...especially extra code which may not be exercised often.

## Setting Up the Project

I'm starting with a directory structure like this:

```console
.nmap-api-server
├── go.mod
├── go.sum
├── main.go
└── nmapserver
    ├── go.mod
    ├── go.sum
    ├── handlers.go
    ├── handlers_test.go
    └── server.go
```

I've run `go mod init` in the root directory and the `nmapserver` directory. In order to refer to `nmapserver` from inside the `main.go` easily, I've added a few lines to my `go.mod` file:

```console
require nmap-api-server.go/nmapserver

require (
    [... other requirements...]
)

replace nmap-api-server.go/nmapserver => ./nmapserver
```

We'll worry about what's in the `main.go` in a little while. For now, we'll worry about the three files created in the `nmapserver` directory.

## Setting Up the Server

In order to get started writing our tests, we need a few things at a bare minimum. We need to bind some routes to some functions, and we need to return an instance of the `fiber.App` server back to whatever function requests a new server, that will be our tests for now, but eventually the `main.go` file in the root of the project.

### server.go

The `server.go` file should look like the following:

```golang
package nmapserver

import "github.com/gofiber/fiber/v2"

func NewServer() *fiber.App {
    srv := fiber.New()
    // eventually we'll handle some
    // server configuration here, but
    // as we have no test that requires it yet,
    // I'm leaving it empty!
    initRoutes(srv)
    return srv
}

func initRoutes(a *fiber.App) {
    a.Get("/", func(c *fiber.Ctx) error {
        return indexHandler(c)
    })
}
```

This bit of code will bind `GET` requests to the `indexHandler` function that will go in the `handlers.go` file:

```golang
package nmapserver

import (
    "github.com/gofiber/fiber/v2"
)

func indexHandler(c *fiber.Ctx) error {
    r := make(map[string]string)
    r["status"] = "ok"
    return c.JSON(r)
}

```

In this case, we're just making a generic JSON response of `{"status": "okay"}` by using a simple `map[string]string`. This is a good enough starting point. Now for the fun stuff, setting up the tests!

## Setting Up handlers_test.go

This is going to be the most complex bit, I'm going to try to break it down into digestible chunks, it'll only get easier after this! In order to write a lot of API unit tests, I find that I need 5 things:

1. **Test Struct**: A uniform test struct that can be leveraged for defining all tests.
2. **Request Func**: A generic function that takes a slice of tests, iterates over them and executes them all, printing the results.
3. **Handler Tests**: The actual tests for the individual handler functions.
4. **Stub Functions**: You will need functions that can return output as if it were your real function when it's not practical to, for example, you can't run `nmap` as part of your TDD as it's not practical, safe, or likely to give you the desired results of your target environment.
5. **Output Validation Functions**: The response your test gets may be complex and you will likely need to parse the output carefully.

In order to get our test running, we need to at least do the first 3. Go is very easy to refactor as you need additional fields, so don't guess about what you'll need, only implement what you actually need right now.

### The Test Struct

For now, the test struct is quite simple:

```golang
type unitTestData struct {
    description          string      // description of the test
    route                string      // route that we're testing
    method               string      // request method being tested
    expectedResponseCode int         // the response code the server should send
    expectedResponseData string      // the data we expect to see in the response
    postData             interface{} // data we send to the server
}
```

This should allow us to define a simple test to ensure that when we make a `GET` request to `/` we get back back `{"status": "okay"}`. I've added `postData` as an `interface{}` for now because we're not actually going to post any data to our end point, but so that both a failing and passing test can be demonstrated, I'll show the code for making a `GET` request and a `POST` request.

### The TestIndexHandler function

The `TestIndexHandler` function is what is actually going to exercise the code in the `handlers.go` file. We'll start with just a simple `GET` request:

```golang
func TestIndexHandler(t *testing.T) {
    tests := []unitTestData{
        {
            description:          "Test GET request",
            route:                "/",
            method:               "GET",
            expectedResponseCode: http.StatusOK,
            expectedResponseData: "{\"status\":\"ok\"}",
        },
    }
    app := NewServer()
    t.Log("====> TESTING DEFAULT ROUTE: /")
    doRequests(t, app, tests)
}
```

In this example I have defined one simple unit test, this is testing that a `GET` request will return a `200 OK` response and the body will contain `{"status": "ok"}` -- in this way, your individual tests are just convenient wrappers around a set of rules for your API. You'll notice there's one function we have not yet implemented, that's the `doRequests()` function.

### The doRequests function

In order to actually execute the tests, http requests must be made against the app. In order to do that, I have written the `doRequests` function as follows:

```golang
func doRequests(t *testing.T, app *fiber.App, tests []unitTestData) {
    for _, test := range tests {
        t.Logf("======> %s", test.description)
        req := httptest.NewRequest(test.method, test.route, nil)
        resp, _ := app.Test(req, -1)
        // Verify the response code is what is expected
        if resp.StatusCode != test.expectedResponseCode {
            t.Errorf("Expected response code %d, got %d", test.expectedResponseCode, resp.StatusCode)
        }
        t.Logf("========> response status: %d", resp.StatusCode)
        // Verify the body content is what is expected
        b, err := io.ReadAll(resp.Body)
        if err != nil {
            t.Errorf("Got error decoding resp.body: %s", err.Error())
        }
        // just do a simple string compare for now
        if string(b) != test.expectedResponseData {
            t.Errorf("Expected a body of:\n%s\ninstead got:\n%s", test.expectedResponseData, string(b))
        }
        t.Logf("========> response body:\n%s", string(b))
    }
}
```

This is a very straight forward function which takes our App and our test data, executes the described steps against the app and verify the results. If you run this test now with `go test -v` you might notice a problem:

```console
go test -v
=== RUN   TestIndexHandler
    handlers_test.go:42: ====> TESTING DEFAULT ROUTE: /
    handlers_test.go:49: ======> Test GET request
    handlers_test.go:68: ========> response status: 200
    handlers_test.go:76: Expected a body of:
        {"status":"ok"}
        instead got:
        {"status":"okay"}
    handlers_test.go:78: ========> response body:
        {"status":"okay"}
--- FAIL: TestIndexHandler (0.00s)
FAIL
exit status 1
FAIL    nmap-api-server/nmapserver    0.003s
```

In this case change the line in `handlers.go` from `r["status"] = "okay"` to `r["status"] = "ok"` and re-run the test to verify it passes:

```console
go test -v
=== RUN   TestIndexHandler
    handlers_test.go:42: ====> TESTING DEFAULT ROUTE: /
    handlers_test.go:49: ======> Test GET request
    handlers_test.go:68: ========> response status: 200
    handlers_test.go:78: ========> response body:
        {"status":"ok"}
--- PASS: TestIndexHandler (0.00s)
PASS
```

## Adding Tests

### Adding POST Requests

The first test I'll add is just a `POST` request to the main function. It should be fine to add this as we should get a method not allowed response as we did not bind `/` to a handler that supports `POST` requests in the `server.go` file. To test that, I've updated the tests struct in the `TestIndexHandler` function as follows:

```golang
    tests := []unitTestData{
        {
            description:          "Test GET request",
            route:                "/",
            method:               "GET",
            expectedResponseCode: http.StatusOK,
            expectedResponseData: "{\"status\":\"ok\"}",
        },
        {
            description:          "Test POST request",
            route:                "/",
            method:               "POST",
            expectedResponseCode: http.StatusMethodNotAllowed,
            expectedResponseData: "Method not allowed.",
        },
    }
```

For the `expectedResponseCode`, I expect that using the wrong method should return a `405` error. For the text, I'm not actually sure what it will return without running, but I expect it will not be `JSON` as our test is not setting the `application/JSON` header type.

Running the test reveals this isn't quite right:

```console
go test -v
=== RUN   TestIndexHandler
    handlers_test.go:42: ====> TESTING DEFAULT ROUTE: /
    handlers_test.go:49: ======> Test GET request
    handlers_test.go:68: ========> response status: 200
    handlers_test.go:78: ========> response body:
        {"status":"ok"}
    handlers_test.go:49: ======> Test POST request
    handlers_test.go:68: ========> response status: 405
    handlers_test.go:76: Expected a body of:
        Method not allowed.
        instead got:
        Method Not Allowed
    handlers_test.go:78: ========> response body:
        Method Not Allowed
--- FAIL: TestIndexHandler (0.00s)
FAIL
exit status 1
```

I will simply take the output under `instead got` and paste that into my test such that the expected output field now reads `Method Not Allowed` -- capitalized, no period:

```console
=== RUN   TestIndexHandler
    handlers_test.go:42: ====> TESTING DEFAULT ROUTE: /
    handlers_test.go:49: ======> Test GET request
    handlers_test.go:68: ========> response status: 200
    handlers_test.go:78: ========> response body:
        {"status":"ok"}
    handlers_test.go:49: ======> Test POST request
    handlers_test.go:68: ========> response status: 405
    handlers_test.go:78: ========> response body:
        Method Not Allowed
--- PASS: TestIndexHandler (0.00s)
PASS
```

You'll notice this method keeps very clear exactly what's going on in our test, the output is concise and lets us know exactly what step we were on when any failure happensed.