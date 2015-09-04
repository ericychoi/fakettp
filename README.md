Fakettp
--------

Fakettp is an http debugging proxy allowing you to easily set up responses for http endpoints while testing. Fakettp allows you to control the following:
  - response code
  - response time
  - response headers
  - response body

You can set global responses, or you can pass in individual endpoints where those responses apply, and proxy all other endpoint requests to the original service. This gives you the ability to test your system against the other service and simulate an error on a single endpoint while the rest of the service works normally.

Use Case
--------

What inspired this repo!: We had Service A calling Service B's endpoints internally. We needed some of the endpoints to do their job, while for one of them we needed to verify how Service A behaved when it returned different content and status codes. After using this tool, our QA was able to verify all the edge-cases they were interested in!

Sample Usage
------------

From the source or compiled binary, just set the response you want (note that you can pass in multiple headers to be returned):


Hyjac a single endpoint and proxy all other calls:
```
go run main.go -proxy_host http://example.com -proxy_port 9092 -hyjack /api/user -port 5555 -code 418 -body "I'm a teapot"
```

We can also use a config file instead (more on this below!):
```
go run main.go -config my.conf
```

Return the same data for all calls to this service:
```
$ go run main.go -port 5555 -code 201 -header 'Content-Type: application/json' -header 'Cache-Control: max-age=3600' -body '{"json":true}'
```

The above setup would result in the following response from any path on `localhost:5555`:
```
$ curl -v localhost:5555/foo/bar/raz -d '{"foo":"bar"}'
*   Trying 127.0.0.1...
* Connected to localhost (127.0.0.1) port 5555 (#0)
> POST /foo/bar/raz HTTP/1.1
> Host: localhost:5555
> User-Agent: curl/7.43.0
> Accept: */*
> Content-Length: 13
> Content-Type: application/x-www-form-urlencoded
>
* upload completely sent off: 13 out of 13 bytes
< HTTP/1.1 201 Created
< Cache-Control: max-age=3600
< Content-Type: application/json
< Date: Wed, 02 Sep 2015 16:17:48 GMT
< Content-Length: 13
<
* Connection #0 to host localhost left intact
{"json":true}
```

Config File
-----------

When passing command line flags, you are limited to either hyjacking all requests or only requests to a single endpoint. With a config file, you can specify multiple routes to behave differently.

Sample Config:
```json
{
    "proxy_host": "http://0.0.0.0",
    "proxy_port": 9092,
    "port": 5000,
    "fakes": [
        {
            "hyjack": "/api/settings.json",
            "code": 500
        },
        {
            "hyjack": "/api/users.json",
            "methods": [
                "GET",
                "POST"
            ],
            "body": "{\"json\":true}",
            "code": 201,
            "headers": [
                "Content-Type: application/json",
                "Cache-Control: max-age=3600"
            ],
            "time": "1m3s15ms"
        }
    ]
}
```

Sample Logs
-----------

Logs show the requested URI, if the request was hyjacked or proxied, and shows a request id at the beginning of log lines dealing with a request.

```
2015/09/02 14:10:22 starting on port :5555
[1dfc34e] 2015/09/02 14:10:23 new request /api/user/get.json
[1dfc34e] 2015/09/02 14:10:23 proxying request
[4666fb4] 2015/09/02 14:10:36 new request /api/functions.json
[4666fb4] 2015/09/02 14:10:36 hyjacking route /api/functions.json (10ms)
[4666fb4] 2015/09/02 14:10:36 setting header Content-Type:application/json
[4666fb4] 2015/09/02 14:10:36 setting header Cache-Control:max-age=3600
```