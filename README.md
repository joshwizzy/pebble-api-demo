# Exploring the Pebble HTTP API

This tutorial explores the Pebble API by following these steps:

* Inspect the source code to figure out the expected payload for the various endpoints
* Write a simple service which we use to test the API.
* Use curl to make requests to the API and observe that we are able to manage the service using the HTTP API.

We explore only a couple of endpoints to demonstrate how to manage services using layers.

# Running the daemon

To run the Pebble daemon, set the `$PEBBLE` environment variable and use the `pebble run` sub-command, something like this:

```shell
$ mkdir ~/pebble
$ export PEBBLE=~/pebble
$ go run ./cmd/pebble run
2023-09-21T13:58:00.564Z [pebble] Started daemon.
2023-09-21T13:58:00.570Z [pebble] POST /v1/services 3.142909ms 400
2023-09-21T13:58:00.572Z [pebble] Cannot start default services: no default services
```


## Using Curl to hit the API

You can use [curl](https://curl.se/) in unix socket mode to hit the Pebble API:

```sh
$ curl --unix-socket ~/pebble/.pebble.socket 'http://localhost/v1/services' |jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    59  100    59    0     0   6846      0 --:--:-- --:--:-- --:--:--  7375
{
  "type": "sync",
  "status-code": 200,
  "status": "OK",
  "result": []
}

```

## Endpoints

We will explore adding adding and updating a service using layers. 
The endpoints we are interested in are:

* `/v1/layers` for adding layers
* `/v1/services` for viewing service status and updating services 

### `/v/layers`

Looking at the [Layers Handler Code](https://github.com/canonical/pebble/blob/b1da4ddd857367cc4f2077e375923bcd975e8390/internals/daemon/api_plan.go#L45) we see that the expected payload is

```go
var payload struct {
		Action  string `json:"action"`
		Combine bool   `json:"combine"`
		Label   string `json:"label"`
		Format  string `json:"format"`
		Layer   string `json:"layer"`
	}
```

`Action` has a single expected value `add`

`Label` is mandatory

`Format` has a single expected value `yaml`

`Layer` is a yaml encoded string which is expected to be a conform to the layer definition

#### `/v1/services`

Looking at the [Services Handler Code](https://github.com/canonical/pebble/blob/b1da4ddd857367cc4f2077e375923bcd975e8390/internals/daemon/api_services.go#L62 )  we see that the expected payload is of the form 

```go
var payload struct {
		Action   string   `json:"action"`
		Services []string `json:"services"`
	}
```

For this tutorial we will use the `replan`  action to reload the service after adding layers 


## Sample service

We will use a simple service written in `Go` to explore the API. 
The service is a simple Hello, World server that listens on a port that is configurable using an environment variable.
It defaults to listening on port `8080` if the `PORT` environment variable is not set.
In your home folder create a file named `server.go` with the following content.

``` Go
package main

import (
        "context"
        "fmt"
        "log"
        "net/http"
        "os"
        "os/signal"
        "time"
)

func main() {
        mux := http.NewServeMux()
        mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
                fmt.Fprint(w, "Hello, world!")
        })

        port := os.Getenv("PORT")
        if port == "" {
                port = "8080"
        }
        srv := http.Server{
                Addr:    ":" + port,
                Handler: mux,
        }

        go func() {
                log.Printf("listening on port: %s\n", port)
                srv.ListenAndServe()
        }()

        ch := make(chan os.Signal, 1)
        signal.Notify(ch, os.Interrupt)
        <-ch
        log.Println("received interrupt")

        ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
        defer cancel()
        srv.Shutdown(ctx)

}
```

Build the code
`go build -o server server.go`

Run the compiled binary

```shell
$ ./server
2023/09/21 17:18:38 listening on port: 8080
```

In another terminal confirm that the server returns the expected response
```
$ curl localhost:8080
Hello, world!
```

Then stop the service `ctrl+c`

## Adding a layer for the service

Add the following content to a yaml file named `hello.yaml`. Replace `ubuntu` with your home folder
``` yaml
summary: Hello World

description: |
    A simple hello world layer.

services:
    srv1:
        override: replace
        startup: enabled
        command: /home/ubuntu/server
```

Use `jq` to quote/escape the yaml content so that we can use it as the `JSON` payload value for the `layer` field

```shell
jq --raw-input --slurp < hello.yaml 
"summary: Hello World\n\ndescription: |\n    A simple hello world layer.\n\nservices:\n    srv1:\n        override: replace\n        startup: enabled\n        command: /home/ubuntu/server\n
```

Create a `hello.json` file with the following content
```json
{
    "action": "add",
    "label": "001-base-layer",
    "format": "yaml",
    "layer": "summary: Hello World\n\ndescription: |\n    A simple hello world layer.\n\nservices:\n    srv1:\n        override: replace\n        startup: enabled\n        command: /home/ubuntu/server\n"
}
```

Make a request to pebble to add the layer
```shell
$ curl --unix-socket ~/pebble/.pebble.socket -X POST --header "Content-Type: application/json" --data @hello.json 'http://localhost/v1/layers'
{"type":"sync","status-code":200,"status":"OK","result":true}
```

View the service status
```shell
$ curl --unix-socket ~/pebble/.pebble.socket 'http://localhost/v1/services' |jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   115  100   115    0     0   7905      0 --:--:-- --:--:-- --:--:--  8846
{
  "type": "sync",
  "status-code": 200,
  "status": "OK",
  "result": [
    {
      "name": "srv1",
      "startup": "enabled",
      "current": "inactive"
    }
  ]
}

```

Make a request to replan
```shell
$ curl --unix-socket ~/pebble/.pebble.socket -X POST --header "Content-Type: application/json" --data '{"action": "replan"}' 'http://localhost/v1/services'
{"type":"async","status-code":202,"status":"Accepted","change":"1","result":null}
```

View the service status again
```shell
$ curl --unix-socket ~/pebble/.pebble.socket 'http://localhost/v1/services' |jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   167  100   167    0     0  12243      0 --:--:-- --:--:-- --:--:-- 12846
{
  "type": "sync",
  "status-code": 200,
  "status": "OK",
  "result": [
    {
      "name": "srv1",
      "startup": "enabled",
      "current": "active",
      "current-since": "2023-09-21T17:30:42.414778042+03:00"
    }
  ]
}

```

Confirm that the service is running and returns the expected response
```shell
$ curl localhost:8080
Hello, world!
```

## Updating the service

Lets update the service to listen on a different port. Modify the `hello.yaml` file with this content
```yaml
summary: Hello World

description: |
    A simple hello world layer.

services:
    srv1:
        override: replace
        startup: enabled
        command: /home/ubuntu/server
        environment:
            PORT: 8081
```

Then encode the yaml
```shell
$ jq --raw-input --slurp < hello.yaml 
"summary: Hello World\n\ndescription: |\n    A simple hello world layer.\n\nservices:\n    srv1:\n        override: replace\n        startup: enabled\n        command: /home/ubuntu/server\n        environment:\n            PORT: 8081\n"
```

Update the `label` and `layer` values in the `hello.json` file
``` json
{
    "action": "add",
    "label": "002-change-port",
    "format": "yaml",
    "layer": "summary: Hello World\n\ndescription: |\n    A simple hello world layer.\n\nservices:\n    srv1:\n        override: replace\n        startup: enabled\n        command: /home/ubuntu/server\n        environment:\n            PORT: 8081\n"
}
```

Add the layer
```shell
$ curl --unix-socket ~/pebble/.pebble.socket -X POST --header "Content-Type: application/json" --data @hello.json 'http://localhost/v1/layers'
{"type":"sync","status-code":200,"status":"OK","result":true}
```

Then replan
```shell
$ curl --unix-socket ~/pebble/.pebble.socket -X POST --header "Content-Type: application/json" --data '{"action": "replan"}' 'http://localhost/v1/services'
{"type":"async","status-code":202,"status":"Accepted","change":"2","result":null}
```

View the service status again
```shell
$ curl --unix-socket ~/pebble/.pebble.socket 'http://localhost/v1/services' |jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   167  100   167    0     0  26647      0 --:--:-- --:--:-- --:--:-- 55666
{
  "type": "sync",
  "status-code": 200,
  "status": "OK",
  "result": [
    {
      "name": "srv1",
      "startup": "enabled",
      "current": "active",
      "current-since": "2023-09-21T17:40:12.584316153+03:00"
    }
  ]
}

```

Confirm that the service is no longer rechable on port `8080`
```shell
$ curl localhost:8080
curl: (7) Failed to connect to localhost port 8080 after 0 ms: Connection refused
```


Confirm that the service is running on port `8081` after the update
```sh
$ curl localhost:8081
Hello, world!
```



## Conclusion

We have explored a subset of the Pebble API. You can explore the full API further by inspecting the [code](https://github.com/canonical/pebble/blob/master/internals/daemon/api.go)
Happy Hacking!