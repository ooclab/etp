# etp

ETP (Easy Transport Protocol) is copy from HTTP, support simple message route & dispatch.

## Example

```go
package main

import (
	"fmt"

	"github.com/ooclab/etp"
)

var routes = []etp.Route{
	{"GET", "/zen", zenHandler},
	{"POST", `/ping/([\-0-9a-fA-F]+)`, pingHandler},
}

func zenHandler(w etp.ResponseWriter, req *etp.Request) {
	w.WriteHeader(etp.StatusOK)
	w.Write([]byte("KISS"))
}

func pingHandler(w etp.ResponseWriter, req *etp.Request) {
	w.WriteHeader(etp.StatusOK)
	etp.WriteJSON(w, 200, etp.H{"ping": "pong"})
}

func request(in, out chan []byte, method, url string, body []byte) (*etp.Response, error) {
	req, err := etp.NewRequest(method, url, body)
	if err != nil {
		return nil, err
	}

	in <- req.Bytes()
	respPayload := <-out

	return etp.ReadResponse(respPayload)
}

func main() {
	in := make(chan []byte)
	out := make(chan []byte)
	quit := make(chan bool)

	router := etp.NewRouter()
	router.AddRoutes(routes)

	go func() {
		var reqPayload []byte
		for {
			// wait a request message
			select {
			case reqPayload = <-in:
				if reqPayload == nil {
					fmt.Println("reqPayload is nil ")
					return
				}
			case <-quit:
				fmt.Println("got quit")
				return
			}

			// read request struct
			req, err := etp.ReadRequest(reqPayload)
			if err != nil {
				fmt.Printf("bad request: %s\n", err)
				continue
			}

			// prepare to dispatch message
			w := etp.NewResponseWriter()
			err = router.Dispatch(w, req)
			if err != nil {
				fmt.Printf("handle error: %s\n%s\n", err, reqPayload)
				w.WriteHeader(etp.StatusBadRequest)
			}

			// send response
			out <- w.Bytes()
		}
	}()

	// client: sent request
	var resp *etp.Response
	var err error

	resp, err = request(in, out, "GET", "/zen", nil)
	if err != nil {
		fmt.Printf("request failed: %s\n", err)
	} else {
		fmt.Printf("GET /zen\n%#v\n", resp)
	}

	close(quit)
	close(in)
	close(out)
}
```

run:

```
GET /zen
&etp.Response{Status:"200 OK", StatusCode:200, Proto:"ETP/1.0", ProtoMajor:1, ProtoMinor:0, PathArgs:[]string(nil), QueryArgs:map[string][]string(nil), Header:http.Header{"Content-Length":[]string{"4"}}, Body:[]uint8{0x4b, 0x49, 0x53, 0x53}, ContentLength:4}
```
