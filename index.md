# Using Test Driven Development in Go to bind services together

One of the most common problems I've run into in the enterprise environment is binding some ancient service to an API for easy consumption by their fancy new React SPA as they attempt to lift and shift old applications from a traditional cludge of scripts and potentially monolithic installation into a nice decomposed application.

In this post I'll walk you through how I have tackled this problem dozens of times over the last few years using an example that would be extremely common.

## The Problem

In this hypothetical scenario we're going to build a tool which can scan VLANs by shelling out and using `nmap` from within a blessed host. In most environments, just running an `nmap` scan against a bunch of hosts to learn things like "did all my VMs become responsive after deployment?" is hopefully going to set off an IDS. In this example we have a blessed host which nobody will care about nmap traffic coming from, we need to write well tested code which will give SecOps the confidence to let us deploy this into our environment to help increase our automation's visibility :)

## A Word About Test Driven Development

In the book "Cloud Native Go" it is emphasized that in order for your tests to drive your development, you must first write a broken test and then get your test to pass by writing the code required to complete the test successfully. More importantly though is what ends up happening when you do this: you write *only the code you need -- nothing more.* As I walk through how I'd tackle this problem space, consider how if we were not writing tests we might accidentally introduce a lot of logic which is beyond the scope of our requirements. In addition to introducing incomplete code which might be in various states of tested, this increases the attack surface of your API for no reason. While the strictest interpretations of TDD are not something I'm going to walk through here, we're primarily embracing any philosophy that minimizes extra code...especially extra code which may not be exercised often (or possibly ever since it was added).

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

In order to get our test running, we need to at least do the first 3. In my experience, Go is very easy to refactor as you build like this, so it's better to wait to add fields as you need them rather than try to anticipate your future needs. I think modifying your past assumptions is almost always harder than implementing a small new piece of functionality.

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

The first test I'll add is just a `POST` request to the default handler function. It should be fine to add this as I should get a method not allowed response because I did not bind `/` to a handler that supports `POST` requests in the `server.go` file. To test that, I've updated the tests struct in the `TestIndexHandler` function as follows:

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

For the `expectedResponseCode`, I expect that using the wrong method should return a `405` error. For the text, I'm not actually sure what it will return without running it, but I expect it will not be `JSON` as our test is not setting the `application/JSON` header type...

Running the test reveals this isn't quite right, but it's extremely close:

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

Before you've gotten to this point you have hopefully planned out your API such that you know what each route is going to expect and what it's going to return. I find myself spending a lot of time in front of a dry erase board during this process as I shuffle around what should go where.

I'm going to implement the handler for the `/check` route next. This is going to be the most code intensive section and I'm going to be moving between files quickly (as well as adding a few new files). In order to implement this function, I need to do some planning. I am going to do that planning in the form of writing out the list of tests I'd like to put together in plain English such that anyone can read and provide feedback on this process:

1. Test that `GET` requests to `/check` return Method Not Allowed.
2. Test that empty `POST` requests get back a 403 Forbidden.
3. Test that a `POST` request with an invalid passphrase returns a 403 Forbidden.
4. Test that a `POST` request with a valid passphrase but invalid VLAN request returns a Bad Request error.
5. Test that a `POST` request with a valid passphrase and valid VLAN returns back some expected output

The first test is to ensure that routes for which we *have not explicitely* bound a handler to do not suddenly start working, for example, if the underlying library changes it's default behavior at some point in the future, this test would help prevent us from putting out potentially broken code. It's also a great place to start in writing my tests.

Before I can get back to our test, I need to bind the `/check` route and setup it's function. Inside of the `server.go`'s `initRoutes` function, update it so that it looks like this:

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

Now add the `checkHandler` function to `handlers.go` by just copy and pasting the `indexHandler` function then changing it's name:

```golang
func checkHandler(c *fiber.Ctx) error {
    r := make(map[string]string)
    r["status"] = "ok"
    return c.JSON(r)
}
```

Now I can start writing the **failing** tests for this function. It's time to open `handlers_test.go` back up and add a new test for the brand new route.

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

In order to start making this route work the way I want it to, there is a *lot* of implied functionality. Taking it in bite sized chunks, the first thing I need to know is what the client should send to actually trigger a scan. There are so many ways to go about this, I am going to pick about the simplest I can imagine. If this were an application actually being deployed, you'd probably want to investigate in something like JWT for auth, but this will be easier to follow. That being said, I need a new file!

I made a `types.go` file to keep track of various requests, responses, configuration and whatever other type I might end up needing. For now, my `types.go` just contains the following:

```golang
package nmapserver

type VlanScanRequest struct {
    Name           string `json:"name"`
    ScanPassphrase string `json:"scan_password"`
}
```

I know I want the request to contain, at a minimum, a VLAN to return data for and a passphrase so that someone can't just go triggering `nmap` scans in my environment.

There are **lots** of ways to do this! This is a crucial point in development, early on in my Go development career, I would go off the rails here adding all sorts of functionality. The test I'm writing does not require a lot functionality! It simply requires that **empty** `POST` requests return a forbidden with proper JSON. Nothing fancy, here's what I came up with:

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

The next test is to provide an **invalid** passphrase to the endpoint. Because I have not yet added any code to deal with the parsed input from the `BodyParser` adding the test will result in a failing test that I will, again, go fix in code after the test case is added. In `handlers_test.go` I'm adding more to the `tests` slice in the `TestCheckHandler` function. To save space here, I am *only* showing the `unitTestData` element for the **newly added** test:

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

The astute reader might notice something that's happened with this latest run. Because the test harness makes a `POST` request, the second test which was passing is now failing. The reason is because go-fiber's `BodyParser` function didn't return an error, it just didn't unmarshall any values into the `scanRequestData` variable! This is **extremely common** in test driven development, and this iterative cycle is what's going to ensure robust testing of your API endpoints.

### Fixing the second test

In order to fix the second test, I need to check the length of the decoded passphrase value to ensure it's not equal to 0. I've updated the `checkHandler` code as follows:

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

This now ensures that a passphrase is required to be sent along, and if it isn't, the application will return a 403 forbidden. Additionally, the second test is fixed and only the newest test is failing:

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
```

Right after the new variable I also have to update the `NewServer` function.

```golang
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

The above will be inside of each Handler function.

Back in `handlers.go` update the `checkHandler` function to check the passphrase:

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

With this validation in place all 3 tests now pass. 3 of the 5 desired tested behaviors in the outline above are now functioning:

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

### Test 4 - Invalid VLAN

Adding this test implies some new functionality that I need to add to the base application. In order to determine if a VLAN is "valid" or "invalid" I need to decide how administrators of this tool are going to define valid VLANs. If I don't have that functionality then I cannot determine if the value is valid or not; in this way, your tests will end up covering more than you initially expect.

Again, to save space, the new test code is isolated below:

```golang
        {
            description:          "Test POST request with valid credentials and invalid VLAN",
            route:                "/check",
            method:               "POST",
            expectedResponseCode: http.StatusBadRequest,
            expectedResponseData: "{\"error\":\"bad request\",\"status\":\"error\"}",
            postData: VlanScanRequest{
                Name:         "INVALIDVLAN",
                ScanPassword: testScanPassphrase,
            },
        },
```

And if I run the test, this returns a `200 OK` which is considered a failure:

```console
go test -v -run TestCheckHandler
=== RUN   TestCheckHandler
[ ... cutting out other tests ... ]
    handlers_test.go:118: ======> Test POST request with valid credentials and invalid VLAN
    handlers_test.go:135: Expected response code 400, got 200
    handlers_test.go:137: ========> response status: 200
    handlers_test.go:145: Expected a body of:
        {"error":"bad request","status":"error"}
        instead got:
        {"status":"ok"}
    handlers_test.go:147: ========> response body:
        {"status":"ok"}
--- FAIL: TestCheckHandler (0.00s)
FAIL
exit status 1
```

In order for this test to pass, I need to do two things:

1. Create a way to configure VLANs which should be scanned.
2. Read configuration and check if a VLAN is valid.

If this was a real production application that is going to be maintained I would probably want the configuration and cached data to be stored in a database of some kind. As that's not really important for the process I'm trying to demonstrate I will juse use good ol' JSON configuration files read from disk. The only configuration information in the VLAN configuration I need is the name of the VLAN and then the range that will be passed in to `nmap`.

Inside the `nmapserver` directory I have created a new folder called `test_data`. Inside of that folder I have created a file called `test_config.json` that contains the following:

```json
{
    "vlans": [
        {
            "name": "testvlan111",
            "nmap_range": "127.0.0.50-250"
        },
        {
            "name": "testvlan112",
            "nmap_range": "127.0.1.50-250"
        }
    ]
}
```

In order to read this, I need to create a few new types in my `types.go` file:

```golang
type VLAN struct {
    Name      string `json:"name"`
    NmapRange string `json:"nmap_range"`
}

type VlanConfiguration struct {
    VLANs []VLAN `json:"vlans"`
}
```

With these two added types, I've decided to create a new file called `utils.go` -- this file is for things that aren't related to types, handlers or the server setup. For now, all I'm going to do is is stick a function in there to decode the configuration. Heres the only function in my `utils.go` for now:

```golang
func decodeVlanConfiguration(path string) (VlanConfiguration, error) {
    r := VlanConfiguration{}
    fh, err := os.Open(path)
    if err != nil {
        return r, errors.New("Error opening configuration file: " + err.Error())
    }
    decoder := json.NewDecoder(fh)
    err = decoder.Decode(&r)
    if err != nil {
        return r, errors.New("Error decoding JSON: " + err.Error())
    }
    return r, nil
}
```

Now I can decode the configuration, but, like the passPhrase, I need to make the code aware of the path to the configuration file. Just like before, update the `server.go` file to add a variable for this:

```golang
var (
    vlanConf       VlanConfiguration
    scanPassphrase string
)
```

Just like before, the `NewServer` function (also in `server.go`) needs another argument so that the code knows what to decode:

```golang
func NewServer(passphrase, configPath string) *fiber.App {
    server := fiber.New()
    var err error
    vlanConf, err = decodeVlanConfiguration(configPath)
    if err != nil {
        log.Fatal("Failed to decode VLAN Configuration:\n" + err.Error())
    }
    scanPassphrase = passphrase
    initRoutes(server)
    return server
}
```

This change has caused another error in `handlers_test.go` -- just like before. I've updated the variables at the top of the `handlers_test.go` file as follows:

```golang
var (
    testVLANConfigurationPath string = "test_data/test_config.json"
    testScanPassphrase        string = "DONOTSCAN"
)
```

Additionally, anywhere in this file where `app := NewServer(testScanPassphrase)` appears, I have updated to include the new variable:

```golang
app := NewServer(testScanPassphrase, testVLANConfigurationPath)
```

With all of this in place I can now update the `checkHandler` function to check if the VLAN name in the request matches any VLAN name in the configuration. I have updated the `CheckHandler` function inside of the `handlers.go` file so that it now has this code right before it returns an `ok`:

```golang
func checkHandler(c *fiber.Ctx) error {
    [... old checkHandler code...]
    // now verify that the requested VLAN exists in
    // the configuration
    var scanVLAN VLAN
    for _, v := range vlanConf.VLANs {
        if v.Name == scanRequestData.Name {
            scanVLAN = v
        }
    }
    // if there is no valid NMAP range, return a bad request
    if scanVLAN.NmapRange == "" {
        r["status"] = "error"
        r["error"] = "bad request"
        return c.Status(fiber.StatusBadRequest).JSON(r)
    }
    r["status"] = "ok"
    return c.JSON(r)
}
```

This should be enough to put the 4th test case into a passing state:

```console
go test -v -run TestCheckHandler
=== RUN   TestCheckHandler

    [... Previous 3 Tests ...]

    handlers_test.go:120: ======> Test POST request with valid credentials and invalid VLAN
    handlers_test.go:139: ========> response status: 400
    handlers_test.go:149: ========> response body:
        {"error":"bad request","status":"error"}
--- PASS: TestCheckHandler (0.00s)
PASS
```

### Test 5 - A Valid Request

Implementing the code to complete the final test as defined is going to address two new, but very commonly encountered problems:

1. How do I validate a complex response body using this design paradigm?
2. How do I test this route *without* triggering an nmap scan? Is it still useful?

There are many ways to prevent certain parts of your code from executing, or executing differently depending on the environment. One way to address this would be through the use of environment variables, and if I were deploying this, that's probably what I would do...but because that's the most obvious I wanted to share another, less obvious way you can control code execution depending on if you're executing tests or not. Remember the `testScanPassphrase` variable I set at the top of the `handlers_test.go` file? I set that value to `DONOTSCAN` -- I think you can see where I'm going with this.

Before I get to implementing how the `DONOTSCAN` code path branching works I have to actually author the failing test, I can't forget to let testing drive my development, even as things start to become more complex and I see more moving parts supporting my test. There are three more fields that the `unitTestData` type requires in order to be able to parse a complex response.

I'd like the response from the API to contain: name, nmap range scanned, the last scanned date, a list of responsive hosts and the total number of hosts alive. I only care about testing some of these fields for now, for example, there's no reason to test equality for the last scanned field.

I have updated the top of my `handlers_test.go` file to contain the three new fields in the `unitTestData` struct:

```golang
type unitTestData struct {
    description          string           // description of the test
    route                string           // route that we're testing
    method               string           // request method being tested
    expectedResponseCode int              // the response code the server should send
    expectedResponseData string           // the data we expect to see in the response
    postData             interface{}      // data we send to the server
    expectedData         interface{}      // the data that we expect to get back
    responseFunc         testResponseFunc // the function that will validate API output
    runResponseFunc      bool             // run the above function
}
```

The `responseFunc` field actually takes generic type, anything that takes as an arg `[]byte` and `interface{}` and returns a `bool` fulfills the `responseFunc` type. I can use this to write custom parsing for each response type **or** response route. To define this interface, I added the following to the top of my `handlers_test.go` file:

```golang
type testResponseFunc func([]byte, interface{}) bool
```

From here, I can add my 5th and final test case into the slice of tests. Once again, I'll omit the previous cases and only show the most recent:

```golang
        {
            description:          "Test POST request with valid credentials and valid VLAN",
            route:                "/check",
            method:               "POST",
            expectedResponseCode: http.StatusOK,
            postData: VlanScanRequest{
                Name:         "testvlan111",
                ScanPassword: testScanPassphrase,
            },
            responseFunc: validateVLANResponseOutput,
            expectedData: VlanResponse{
                Name:            "testvlan111",
                NmapRange:       "127.0.0.50-250",
                ResponsiveHosts: []string{"127.0.0.50", "127.0.0.51"},
                HostsAlive:      2,
            },
            runResponseFunc: true,
        },
```

For `responseFunc` I provided another function which I have not yet written *because* I need to first add the `VLANResponse` type which the `/check` route should ultimately respond with as per the `expectedData` field.

In the `types.go` file add a type for `VLANResponse`:

```golang
type VlanResponse struct {
    Name            string   `json:"name"`
    NmapRange       string   `json:"nmap_range"`
    LastScannedDate string   `json:"last_scanned_date"`
    ResponsiveHosts []string `json:"responsive_hosts"`
    HostsAlive      int      `json:"hosts_alive"`
}
```

With this type defined, I can go back to the `handlers_test.go` file and the `validateVLANResponseOutput` function.

> **A thought on organizing** -- you might find yourself wondering how to break up the `responseFunc` values. In my experience, have one function per type and leverage the `expectedData` to validate different your various test cases.

The `validateVLANResponseOutput` function:

```golang
func validateVLANResponseOutput(serverResponse []byte, expectedData interface{}) bool {
    var server_data VlanResponse
    err := json.Unmarshal(serverResponse, &server_data)
    if err != nil {
        return false
    }
    if ex_data, ok := expectedData.(VlanResponse); ok {
        var r bool = true
        if server_data.Name != ex_data.Name {
            r = false
        }
        if server_data.NmapRange != ex_data.NmapRange {
            r = false
        }
        if server_data.HostsAlive != ex_data.HostsAlive {
            r = false
        }
        if len(server_data.ResponsiveHosts) != len(ex_data.ResponsiveHosts) {
            r = false
        }
        return r
    } else {
        return false
    }
}
```

Now that this function exists, the `doRequest` function also needs to be modified to reflect the fact that these new fields exist and that I need to do something with them. The end of the function now includes this code:

```golang
        // this was in the previous function
        if err != nil {
            t.Errorf("Got error decoding resp.body: %s", err.Error())
        }
        // This is all new stuff

        // sometimes it doesn't make sense to compare a string, but to
        // ensure that data decodes properly
        if test.runResponseFunc {
            t.Logf("========> Testing equality between response and expectedData")
            // should we run a custom function to test the response
            if !test.responseFunc(b, test.expectedData) {
                t.Errorf("Two values are not equal.\nResponse:%v\nExpected Data:\n%v", string(b), test.expectedData)
            }
        } else {
            // this was the only thing that was done before
            if string(b) != test.expectedResponseData {
                t.Errorf("Expected a body of:\n%s\ninstead got:\n%s", test.expectedResponseData, string(b))
            }
        }
        t.Logf("========> response body:\n%s", string(b))
```

With these changes in place, I can now run the `TestCheckHandler` test again and confirm that the server sends back `{"status": "ok"}` instead if the VLAN data I've defined in the test:

```console
go test -v -run TestCheckHandler
=== RUN   TestCheckHandler
    [ ... Output for Previous 4 Tests ... ]
    handlers_test.go:149: ======> Test POST request with valid credentials and valid VLAN
    handlers_test.go:168: ========> response status: 200
    handlers_test.go:178: ========> Testing equality between response and expectedData
    handlers_test.go:181: Two values are not equal.
        Response:{"status":"ok"}
        Expected Data:
        {testvlan111 127.0.0.50-250  [127.0.0.50 127.0.0.51] 2}
    handlers_test.go:189: ========> response body:
        {"status":"ok"}
--- FAIL: TestCheckHandler (0.00s)
FAIL
exit status 1
```

Before diving in, it's important to consider exactly what will be required to get this test functioning. It should take:

1. Call a function from inside the `checkHandler` which will do any pre-flight checks necessary before doing the scan.
2. This new function should read the scan passphrase to determine if it should shell out and call `nmap` and return the response or, should it return some hard coded data in order to test that the API route is functioning?
3. Parse either the real or simulated output and return it back to the client.

The most logical thing I could come up with was to create a function called `runNmapScan` which will take the IP range from the configuration as it's only argument. It will return back a slice of strings and optionally an error. I've added this code into the `utils.go` file:

```golang
func runNmapScan(ipRange string) ([]string, error) {
    var r []string
    var nmapResponse string
    var stderr string
    var err error
    if scanPassphrase == "DONOTSCAN" {
        nmapResponse = execFakeNmapCommand()
    } else {
        nmapResponse, stderr, err = execCommand("/usr/bin/nmap", "-sn", ipRange)
        if err != nil {
            log.Printf("Got error: %s", stderr)
            return r, err
        }
    }
    // now we need to parse out just the IP addresses from
    // the scan and need to append each IP address to the variable r
    scanner := bufio.NewScanner(strings.NewReader(nmapResponse))
    for scanner.Scan() {
        t := scanner.Text()
        if strings.Contains(t, "Nmap scan report for ") {
            ip := strings.Replace(t, "Nmap scan report for ", "", -1)
            r = append(r, ip)
        }
    }
    return r, nil
}
```

This new function is where I've implemented the `DONOTSCAN` logic, specifically:

```golang
   if scanPassphrase == "DONOTSCAN" {
        nmapResponse = execFakeNmapCommand()
    } else {
        nmapResponse, stderr, err = execCommand("/usr/bin/nmap", "-sn", ipRange)
        if err != nil {
            log.Printf("Got error: %s", stderr)
            return r, err
        }
    }
```

There is another function (not shown) which will execute a system command and return back a string for `STDOUT`, `STDERR` as well as any error captured by Go. The `execFakeNmapCommand` function was created by running `nmap` in the environment I intend to deploy to and then modifying it to match what the test actually expects:

```golang
func execFakeNmapCommand() string {
    return `
Starting Nmap 6.40 ( http://nmap.org ) at 2023-02-27 09:16 PDT
Nmap scan report for 127.0.0.50
Host is up (0.00036s latency).
Nmap scan report for 127.0.0.51
Host is up (0.00030s latency).
Nmap done: 201 IP addresses (167 hosts up) scanned in 1.22 seconds`
}
```

This is effectively a copy/paste from the terminal, and then obviously host `127.0.0.50` or `51` was not really up, this is just to ensure this string output matches what the test case is going to be looking for. It should also be noted that if I was building this for real, there would be much more robust validation on the input and output, as well as test cases around DNS names coming back, that sort of thing. This is not intended to be a **complete** example.

> **But shouldn't there be tests for using `nmap`?** As I am not an `nmap` developer, my philosophy in test driven development is that I cannot test other people's projects. My job is to test that both expected and unexpected inputs input to the API I am authoring returns safe output. It would be the job of, in this case, `nmap` developers to ensure their product is well tested and works well. As a microservice developer, we're effectively acting as an interface between some project and a web API.

I can finally hook everything together for the `/check` route by adding the following code to the bottom of the `checkHandler` function in the `handlers.go` file:

```golang
    // all checks seem good, so run the nmap scan
    hostsUp, err := runNmapScan(scanVLAN.NmapRange)
    if err != nil {
        log.Printf("nmap scan failed with following error:\n%v", err)
        r["status"] = "error"
        r["error"] = "error running scan, see server logs for details"
        return c.Status(fiber.StatusInternalServerError).JSON(r)
    }
```

One very important note on the above, because you are shelling out and running system commands, just like you should not pass unsanitized input from the client to any applications you're calling, so you also **should NOT** pass back any log data which may have been returned back to the client. This could reveal critical configuration details, always just return a generic message to the client.

Finally, I used the value of `hostsUp` and other details from the request to build a proper `VlanResponse` and return that back to the client:

```golang
    // now build the VlanResponse struct which is also used to
    // cache the data to json files on disk
    vlanResponse := VlanResponse{
        Name:            scanVLAN.Name,
        NmapRange:       scanVLAN.NmapRange,
        LastScannedDate: time.Now().Format("2006-01-02 15:04:05"),
        ResponsiveHosts: hostsUp,
        HostsAlive:      len(hostsUp),
    }
    return c.JSON(vlanResponse)
```

I can now re-run the test and see that everything passes:

```console
go test -v -run TestCheckHandler
=== RUN   TestCheckHandler
    handlers_test.go:116: ====> TESTING ROUTE: /check
    handlers_test.go:149: ======> Test GET request
    handlers_test.go:168: ========> response status: 405
    handlers_test.go:189: ========> response body:
        Method Not Allowed
    handlers_test.go:149: ======> Test POST request with no credentials
    handlers_test.go:168: ========> response status: 403
    handlers_test.go:189: ========> response body:
        {"error":"forbidden","status":"error"}
    handlers_test.go:149: ======> Test POST request with invalid credentials
    handlers_test.go:168: ========> response status: 403
    handlers_test.go:189: ========> response body:
        {"error":"forbidden","status":"error"}
    handlers_test.go:149: ======> Test POST request with valid credentials and invalid VLAN
    handlers_test.go:168: ========> response status: 400
    handlers_test.go:189: ========> response body:
        {"error":"bad request","status":"error"}
    handlers_test.go:149: ======> Test POST request with valid credentials and valid VLAN
    handlers_test.go:168: ========> response status: 200
    handlers_test.go:178: ========> Testing equality between response and expectedData
    handlers_test.go:189: ========> response body:
        {"name":"testvlan111","nmap_range":"127.0.0.50-250","last_scanned_date":"2023-02-28 17:37:09","responsive_hosts":["127.0.0.50","127.0.0.51"],"hosts_alive":2}
--- PASS: TestCheckHandler (0.00s)
PASS
```

### Next Steps

There are more test cases that should be written, I think 5 is enough to demonstrate some of the ways that I have tackled these problems over the years and continue to do so. Using this modular method makes it incredibly easy to keep adding test cases as bugs and new unexpected output are discovered, or even hooking your application up to some automated fuzzing using the concepts outlined here would be trivial.

This sort of application is very easily deployed in a dockerfile, as part of a Kubernetes deployment, or even just building a Go binary and running it on bare metal. It's so flexible that this has become my conceptual glue for addressing building APIs in this space.

## Wrapping Up

As I said at the beginning, I would eventually get back to the `main.go` file. There's a couple of interesting things to know about this file. The first is how the `nmapserver` is accessed. At the top of this article I noted that the `go.mod` has in it the following:

```console
require nmap-api-server.go/nmapserver
replace nmap-api-server.go/nmapserver => ./nmapserver
```

This indicates that the import should reference `nmap-api-server.go/nmapserver`. With that knowledge and knowing there are two environment variables to pass to the `nmapserver`:

```golang
package main

import (
    "fmt"
    "log"
    "os"

    "nmap-api-server.go/nmapserver"
)

func main() {
    port := os.Getenv("VLANSERVER_PORT")
    if len(port) == 0 {
        port = "8080"
    }
    scanPass := os.Getenv("SCAN_PASSPHRASE")
    if len(scanPass) == 0 {
        log.Println("SCAN_PASSPHRASE must be a defined environment variable")
        os.Exit(1)
    }
    config := os.Getenv("CONFIG_PATH")
    if len(config) == 0 {
        log.Println("CONFIG_PATH must point to a valid configuration file")
        os.Exit(1)
    }
    server := nmapserver.NewServer(config, scanPass)
    log.Fatalln(server.Listen(fmt.Sprintf(":%v", port)))
}
```

In this way the `main.go` file is very generic and just revolves around collecting the information you need to get from the environment. Using environment variables can be a good idea for most configuration as it is easy to integrate with container orchestration, systemd unit files, or even just running the app in a screen session. The flexibility makes this a very reliable approach for most uses. This is **NOT** a good idea for storing/reading passwords. Please use proper secret management for this purpose.