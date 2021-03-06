# Go APNS Mock

APNS Mock is a configurable emulator of Apple Push Notification HTTP/2 service written in Go.
An embeddable server as well as a standalone command line utility are provided.

[![Build Status](https://travis-ci.org/baobabus/go-apnsmock.svg?branch=master)](https://travis-ci.org/baobabus/go-apnsmock)
[![GoDoc](https://godoc.org/github.com/baobabus/go-apnsmock/apns2mock?status.svg)](https://godoc.org/github.com/baobabus/go-apnsmock/apns2mock)

## Features

- Emulation of new Apple Push Notification service over HTTP/2 protocol
- Configurable connection handling options (stream concurrency, latency, etc.)
- Emulation of token-based authentication (JWT)
- Emulation of TLS client certificate-based authentication (coming)
- Preconfigured set of request handling scenarios including many deterministic failure cases
- Support for custom request handling scenarios
- Supports Go 1.7 and later

## Handling of mock communications

The ability to mock various communication scenarios is the primary goal of APNS Mock package.
The following comms-related parameters can be customized to help with testing of your APNS clients.

- Connection delay - simulate long connection delays or set it to 0
- Response time - configure how long the server takes to respond to a request
- Maximum number of open connections - test how well your client handles refused connections
- Maximum number of concurrent HTTP/2 streams - test how your client handles low and high numbers

In addition to the above, you can programmaticaly instruct the mock server to become unavailable
or to resume normal processing at any point so that you can test your client's handling of such scenarios.


## Request validation

The following validation is performed by the default request handler:

- Request methods other than POST return 405, "MethodNotAllowed"
- Non-hexadecimal device tokens return 400, "BadDeviceToken"
- Missing or incorrect authorization header returns 403, "MissingProviderToken"
- Malformed JWT headres/claims return 403, "InvalidProviderToken"
- Tokens with incorrct signing algorithm return 403, "InvalidProviderToken"
- Missing request body returns 400, "PayloadEmpty"
- Invalid UUID format in APNS Id returns 400, "BadMessageId"
- Priority that is not empty, 5 or 10 returns 400, "BadPriority"
- Empty topic returns 400, "MissingTopic"
- Collapse Ids longer than 64 return 400, "BadCollapseId"
- Expiration date that cannot be parsed returns 400, "BadExpirationDate"
- Expired tokens return 403, "ExpiredProviderToken"

Or use AllOkayHandler if request validation is not desired. 

## Precofigured failure scenarios

The following scenarios produce mock rejection responses:

- Device tokens starting with '1' return 400, "BadDeviceToken"
- Device tokens starting with '2' return 410, "Unregistered"
- Device tokens starting with the same letter/digit as APNS topic return 400, "DeviceTokenNotForTopic"
- Topics starting with 'd' return 400, "TopicDisallowed"
- Team IDs (JWT "iss" claims) starting with '1' return 403, "InvalidProviderToken"

Setting you request handler to AllOkayHandler will turn off mock push rejections. 


## Command line

`go-apnsmock` is a command line tool that can be used to run a standalone APNS emulator.

Usage:

```
go-apnsmock <flags>

Flags:

  -addr address
    	network address to serve on (default "127.0.0.1:8443")
  -allok
    	if allok is true, server will respond with 200 status to all requests
  -cert path
    	path to server TLS certificate (default "certs/server.crt")
  -conn-delay time
    	amount of time by which client connect attempts should be delayed (default 100ms)
  -conns number
    	maximum number of concurrent HTTP/2 connections (default 5)
  -key path
    	path to TLS certificate key (default "certs/server.key")
  -resp-delay time
    	amount of time by which responses should be delayed (default 5ms)
  -streams number
    	number of concurrent HTTP/2 streams (default 500)
  -verbose
    	if true, verbose enables http2 verbose logging
```

## Embedding in automated tests

Instances of `apns2mock.Server` can be easily embedded in automated tests.
Any number of server instances can be concurrently instantiated while providing complete isolation from each other.

Unit test example

```go
package apns2

import (
	"strings"
	"testing"

	"github.com/baobabus/go-apnsmock/apns2mock"
)

func TestRoundtrip(t *testing.T) {
	s, err := apns2mock.NewServer(apns2mock.NoDelayCommsCfg, apns2mock.AllOkayHandler, apns2mock.AutoCert, apns2mock.AutoKey)
	if err != nil {
		t.Fatal(err)
	}
	defer s.Close()

	// client is preconfigured for communication with the server
	client := s.Client()
	url := s.URL + apns2mock.RequestRoot
	cont := "application/json; charset=utf-8"

	// Expecting to get 200 back
	resp, err := client.Post(url, cont, strings.NewReader("{}"))
	if err != nil {
		t.Fatal(err)
	}
	if resp == nil {
		t.Fatal("Should have gotten a response")
	}
	if resp.StatusCode != 200 {
		t.Fatal("Should have gotten status 200")
	}

	s.BecomeUnavailable(503, "Shutdown")

	// Now expecting 503
	resp, err = client.Post(url, cont, strings.NewReader("{}"))
	if err != nil {
		t.Fatal(err)
	}
	if resp == nil {
		t.Fatal("Should have gotten a response")
	}
	if resp.StatusCode != 503 {
		t.Fatal("Should have gotten status 503")
	}
}
```

## License

The MIT License (MIT)

Copyright (c) 2017 Aleksey Blinov

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
