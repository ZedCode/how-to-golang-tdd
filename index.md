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
}
```

This should allow us to define a simple test to ensure that when we make a `GET` request to `/` we get back back `{"status": "okay"}`.

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

## Adding a test for a new route

Before you've gotten to this point you have hopefully planned out your API such that you know what each route is going to expect and what it's going to return. I find myself spending a lot of time in front of a dry erase board during this process as I shuffle around what should go where as my little API microservice takes shape.

I'm going to implement the handler for the `/check` route, this is going to be the most code intensive section and I'm going to be moving between files quickly (as well as adding a few new files). In order to implement this function, I need to do some planning. I am going to do that planning in the form of simply writing out the list of tests I'd like to put together in plain English such that anyone can read and provide feedback on this process:

1. Test that `GET` requests to `/check` return Method Not Allowed.
2. Test that empty `POST` requests get back a 403 Forbidden.
3. Test that a `POST` request with an invalid passphrase returns a 403 Forbidden.
4. Test that a `POST` request with a valid passphrase but invalid VLAN request returns a Bad Request error.
5. Test that a `POST` request with a valid passphrase and valid VLAN returns back some expected output

The first test is to ensure that routes for which we *have not explicitely* bound a handler to do not suddenly start working, for example, if the underlying library changes it's default behavior at some point in the future, this test would help prevent us from putting out potentially broken code. It's also a great place to start in writing our tests.

Before we can get back to our test, let's bind the `/check` route and setup it's function by copying the `IndexHandler` func. Inside of the `server.go`'s `initRoutes` function, update it so that it looks like this:

```golang
func initRoutes(a *fiber.App) {
    a.Get("/", func(c *fiber.Ctx) error {
        return indexHandler(c)
    })
    a.Post("/check", func(c *fiber.Ctx) error {
        return checkHandler(c)
    })
}
```

Now add the `checkHandler` function to `handlers.go` by just duplicating the `indexHandler`:

```golang
func checkHandler(c *fiber.Ctx) error {
    r := make(map[string]string)
    r["status"] = "ok"
    return c.JSON(r)
}
```

Perfect, now we can start writing the **failing** tests for this function. Move over to `handlers_test.go` and add a new test for the brand new route.

```golang
func TestCheckHandler(t *testing.T) {
    tests := []unitTestData{
        {
            description:          "Test GET request",
            route:                "/check",
            method:               "GET",
            expectedResponseCode: http.StatusMethodNotAllowed,
            expectedResponseData: "Method Not Allowed",
        },
    }
    app := NewServer()
    t.Log("====> TESTING ROUTE: /check")
    doRequests(t, app, tests)
}
```

Now the first test matches the description of the first test above. Because in the `server.go` file I only bound this route to `POST` requests, the first test should "just work" -- the best kind of work:

```console
go test -v -run TestCheckHandler       
=== RUN   TestCheckHandler
    handlers_test.go:105: ====> TESTING ROUTE: /check
    handlers_test.go:112: ======> Test GET request
    handlers_test.go:131: ========> response status: 405
    handlers_test.go:141: ========> response body:
        Method Not Allowed
--- PASS: TestCheckHandler (0.00s)
PASS
```

It's also a great time to point out that you can narrow the scope of the tests you are running with the `-run` flag and then the name of the test you want to execute. Since that worked, add a second test, this time for a POST request by updating the `tests` slice in the `TestCheckHandler` function:

```golang
    tests := []unitTestData{
        {
            description:          "Test GET request",
            route:                "/check",
            method:               "GET",
            expectedResponseCode: http.StatusMethodNotAllowed,
            expectedResponseData: "Method Not Allowed",
        },
        {
            description:          "Test POST request with no credentials",
            route:                "/check",
            method:               "POST",
            expectedResponseCode: http.StatusForbidden,
            expectedResponseData: "{\"error\":\"forbidden\",\"status\":\"error\"}",
        },
    }
```

Running this test results in a `200 OK` -- which, in this case, is a failure:

```console
go test -v -run TestCheckHandler
=== RUN   TestCheckHandler
    handlers_test.go:105: ====> TESTING ROUTE: /check
    handlers_test.go:112: ======> Test GET request
    handlers_test.go:131: ========> response status: 405
    handlers_test.go:141: ========> response body:
        Method Not Allowed
    handlers_test.go:112: ======> Test POST request with no credentials
    handlers_test.go:129: Expected response code 403, got 200
    handlers_test.go:131: ========> response status: 200
    handlers_test.go:139: Expected a body of:
        {"error":"forbidden","status":"error"}
        instead got:
        {"status":"ok"}
    handlers_test.go:141: ========> response body:
        {"status":"ok"}
--- FAIL: TestCheckHandler (0.00s)
FAIL
exit status 1
```

### Putting the "development" in TDD

In order to start making this route work the way I want it to, there is a *lot* of implied functionality. Taking it in bite sized chunks, the first thing I need to know is what the client should send to actually trigger a scan? There are so many ways to go about this, I am going to pick about this simplest. If this were an application actually being deployed, you'd probably want to investigate in something like JWT for auth and not this simple, in-line solution I'm putting here. That being said, I need a new file!

I made a `types.go` file to keep track of various requests, responses, configuration and whatever other type I might end up needing. For now, my `types.go` just contains the following:

```golang
package nmapserver

type VlanScanRequest struct {
    Name           string `json:"name"`
    ScanPassphrase string `json:"scan_password"`
}
```

I know I want the request to contain, at a minimum, a VLAN to return data for and a passphrase so that someone can't just go triggering `nmap` scans in my environment.

There are **lots** of ways to do this, but this is a crucial point where early on, I would go off the rails here. The test I'm writing does not require a lot! It simply requires that an **empty** `POST` requests return forbidden with proper JSON. Nothing fancy, here's what I came up with:

```golang
func checkHandler(c *fiber.Ctx) error {
    r := make(map[string]string)
    scanRequestData := new(VlanScanRequest)
    if err := c.BodyParser(scanRequestData); err != nil {
        r["status"] = "error"
        r["error"] = "forbidden"
        return c.Status(fiber.StatusForbidden).JSON(r)
    }
    r["status"] = "ok"
    return c.JSON(r)
}
```

The test now passes even though we don't yet have any logic around this, that's okay, that's not what this test was for!

```console
go test -v -run TestCheckHandler
=== RUN   TestCheckHandler
    handlers_test.go:105: ====> TESTING ROUTE: /check
    handlers_test.go:112: ======> Test GET request
    handlers_test.go:131: ========> response status: 405
    handlers_test.go:141: ========> response body:
        Method Not Allowed
    handlers_test.go:112: ======> Test POST request with no credentials
    handlers_test.go:131: ========> response status: 403
    handlers_test.go:141: ========> response body:
        {"error":"forbidden","status":"error"}
--- PASS: TestCheckHandler (0.00s)
PASS
```

The next test is to provide an **invalid** passphrase to the endpoint. Because I have not yet added any code to deal with the parsed input from the `BodyParser` adding the test will result in a failing test that I will, again, go fix in code after the test case is added. In `handlers_test.go` I'm adding more to the `tests` slice in the `TestCheckHandler` method. To save space here, I am *only* showing the `unitTestData` element for the *newly added* test:

```golang
        {
            description:          "Test POST request with invalid credentials",
            route:                "/check",
            method:               "POST",
            expectedResponseCode: http.StatusForbidden,
            expectedResponseData: "{\"error\":\"forbidden\",\"status\":\"error\"}",
            postData: VlanScanRequest{
                Name:           "foo",
                ScanPassphrase: "INVALIDSCANPASSWORD",
            },
        },
```

The field `postData` did not exist in the `unitTestData` type, so add that up at the top of the `handlers_test.go` inside the struct definition:

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

Because the `postData` can be anything, I just made it an interface. In this case, I'm putting a `VlanScanRequest` there.

It's at this point that I also have to put in some accounting for POST requests into the `doRequests` function in my `handlers_test.go` file. I've rewritten the function so that it now looks like this:

```golang
func doRequests(t *testing.T, app *fiber.App, tests []unitTestData) {
    for _, test := range tests {
        t.Logf("======> %s", test.description)
        var req *http.Request
        if test.method == "POST" {
            // if POSTing submit data, if the field isn't set this
            // will still work.
            requestByteArray, err := json.Marshal(test.postData)
            if err != nil {
                t.Errorf("Failed to encode request data into byte array:\n%v", err)
            }
            req = httptest.NewRequest(test.method, test.route, bytes.NewBuffer(requestByteArray))
            // if we don't submit this as json it will fail to parse
            req.Header.Add("Content-Type", "application/json")
        } else {
            req = httptest.NewRequest(test.method, test.route, nil)
        }
        resp, _ := app.Test(req, -1)
        if resp.StatusCode != test.expectedResponseCode {
            t.Errorf("Expected response code %d, got %d", test.expectedResponseCode, resp.StatusCode)
        }
        t.Logf("========> response status: %d", resp.StatusCode)
        b, err := io.ReadAll(resp.Body)
        if err != nil {
            t.Errorf("Got error decoding resp.body: %s", err.Error())
        }
        if string(b) != test.expectedResponseData {
            t.Errorf("Expected a body of:\n%s\ninstead got:\n%s", test.expectedResponseData, string(b))
        }
        t.Logf("========> response body:\n%s", string(b))
    }
}
```

This is very similar to the old code except that if it's a `POST` request it'll marshall up the value of `test.postData` and send it to the app. Now that the test harness supports `POST` data I can re-run the test and see what kind of output I get:

```console
go test -v -run TestCheckHandler
=== RUN   TestCheckHandler
    handlers_test.go:107: ====> TESTING ROUTE: /check
    handlers_test.go:114: ======> Test GET request
    handlers_test.go:133: ========> response status: 405
    handlers_test.go:143: ========> response body:
        Method Not Allowed
    handlers_test.go:114: ======> Test POST request with no credentials
    handlers_test.go:131: Expected response code 403, got 200
    handlers_test.go:133: ========> response status: 200
    handlers_test.go:141: Expected a body of:
        {"error":"forbidden","status":"error"}
        instead got:
        {"status":"ok"}
    handlers_test.go:143: ========> response body:
        {"status":"ok"}
    handlers_test.go:114: ======> Test POST request with invalid credentials
    handlers_test.go:131: Expected response code 403, got 200
    handlers_test.go:133: ========> response status: 200
    handlers_test.go:141: Expected a body of:
        {"error":"forbidden","status":"error"}
        instead got:
        {"status":"ok"}
    handlers_test.go:143: ========> response body:
        {"status":"ok"}
--- FAIL: TestCheckHandler (0.00s)
FAIL
exit status 1
```

The astute reader might notice something that's happened with this latest run. Because the test harness makes a `POST` request, the second test which was passing is now failing. This is because go-fiber's `BodyParser` function didn't return an error, it just didn't unmarshall any values into the `scanRequestData` variable! This is **extremely common** in test driven development, and this iterative cycle is what's going to ensure robust testing of your API endpoints.

### Fixing the second test

In order to fix the second test, we need to check the length of the decoded passphrase value to ensure it's not equal to 0. I've updated the `checkHandler` code as follows:

```golang
func checkHandler(c *fiber.Ctx) error {
    r := make(map[string]string)
    scanRequestData := new(VlanScanRequest)
    if err := c.BodyParser(scanRequestData); err != nil {
        r["status"] = "error"
        r["error"] = "forbidden"
        return c.Status(fiber.StatusForbidden).JSON(r)
    }
    if len(scanRequestData.ScanPassphrase) == 0 {
        r["status"] = "error"
        r["error"] = "forbidden"
        return c.Status(fiber.StatusForbidden).JSON(r)
    }
    r["status"] = "ok"
    return c.JSON(r)
}
```

This now ensures that a passphrase is required to be sent along, and if it isn't, the application will return a 403 forbidden. Additionally, only the newest test is failing:

```console
go test -v -run TestCheckHandler
=== RUN   TestCheckHandler
    handlers_test.go:107: ====> TESTING ROUTE: /check
    handlers_test.go:114: ======> Test GET request
    handlers_test.go:133: ========> response status: 405
    handlers_test.go:143: ========> response body:
        Method Not Allowed
    handlers_test.go:114: ======> Test POST request with no credentials
    handlers_test.go:133: ========> response status: 403
    handlers_test.go:143: ========> response body:
        {"error":"forbidden","status":"error"}
    handlers_test.go:114: ======> Test POST request with invalid credentials
    handlers_test.go:131: Expected response code 403, got 200
    handlers_test.go:133: ========> response status: 200
    handlers_test.go:141: Expected a body of:
        {"error":"forbidden","status":"error"}
        instead got:
        {"status":"ok"}
    handlers_test.go:143: ========> response body:
        {"status":"ok"}
--- FAIL: TestCheckHandler (0.00s)
FAIL
exit status 1
```

### Back to Test 3

Now that the second test is passing I need a way to actually check if the provided scanner passphrase is the right one or not. I also need to set the passphrase in a few places. First, I want to make a variable to hold this value. I've chosen to put this in the top of the `server.go` as one would probably expect to find it:

```golang
import (
    "github.com/gofiber/fiber/v2"
)

var (
    scanPassphrase string
)

func NewServer(passphrase string) *fiber.App {
    server := fiber.New()
    scanPassphrase = passphrase
    initRoutes(server)
    return server
}
```

I've added an argument to the `NewServer` function requiring a passphrase. I've then assigned whatever is set there to the `scanPassphrase` variable so that it can be accessed by the various handlers.

This means I need to also update `handlers_test.go` as it is creating an instance of `NewServer`. Because in each handler function I create an instance of the `NewServer` app, it'll be easiest to set a variable at the top of the `handlers_test.go`:

```golang
var (
    testScanPassphrase string = "DONOTSCAN"
)
```

Then, anywhere `NewServer` is called, add this variable:

```golang
app := NewServer(testScanPassphrase)
```

Finally, back in `handlers.go` update the `checkHandler` function to check the passphrase:

```golang
func checkHandler(c *fiber.Ctx) error {
    r := make(map[string]string)
    scanRequestData := new(VlanScanRequest)
    if err := c.BodyParser(scanRequestData); err != nil {
        r["status"] = "error"
        r["error"] = "forbidden"
        return c.Status(fiber.StatusForbidden).JSON(r)
    }
    if len(scanRequestData.ScanPassword) == 0 {
        r["status"] = "error"
        r["error"] = "forbidden"
        return c.Status(fiber.StatusForbidden).JSON(r)
    }
    if scanRequestData.ScanPassword != scanPassphrase {
        r["status"] = "error"
        r["error"] = "forbidden"
        return c.Status(fiber.StatusForbidden).JSON(r)
    }
    r["status"] = "ok"
    return c.JSON(r)
}
```

With this check in place the test now passes, with 3 of the 5 desired tested behaviors functioning:

```console
go test -v -run TestCheckHandler
=== RUN   TestCheckHandler
    handlers_test.go:111: ====> TESTING ROUTE: /check
    handlers_test.go:118: ======> Test GET request
    handlers_test.go:137: ========> response status: 405
    handlers_test.go:147: ========> response body:
        Method Not Allowed
    handlers_test.go:118: ======> Test POST request with no credentials
    handlers_test.go:137: ========> response status: 403
    handlers_test.go:147: ========> response body:
        {"error":"forbidden","status":"error"}
    handlers_test.go:118: ======> Test POST request with invalid credentials
    handlers_test.go:137: ========> response status: 403
    handlers_test.go:147: ========> response body:
        {"error":"forbidden","status":"error"}
--- PASS: TestCheckHandler (0.00s)
PASS
```