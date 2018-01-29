# Stratum

A collection of client and server RPC codecs for interacting with Monero miners and pools.

## Installation

`go get github.com/trey-jones/stratum`

## Usage

[![GoDoc](https://godoc.org/github.com/trey-jones/stratum?status.svg)](https://godoc.org/github.com/trey-jones/stratum)

[Check out the full documentation on godoc](https://godoc.org/github.com/trey-jones/stratum)

There are some differences between JSON RPC 2.0 and (typical?) Monero mining communication. Each end of the connection has some features of a client **and** a server.

`Client` embeds a `rpc.Client` and `Server` embeds `rpc.Server`.  Learn more about their capabilities from the [official documentation.](https://golang.org/pkg/net/rpc)

* Clients receive notifications (especially **jobs**) from the server.
* Servers broadcast notifications to clients, rather than simply responding to calls.
* Method names don't include a service name.

This means usage is a little different than most RPC codecs for net/rpc.
For a more thorough usage example check out the [project that spawned this one.](https://github.com/trey-jones/xmrwasp)

### Client

You can get a client with Dial:

`c, err := stratum.Dial("tcp", "xmrpool.eu:3333")`

If you want to specify a timeout:

`c, err := stratum.DialTimeout("tcp", "xmrpool.eu", 10 * time.Second)`

Since clients can receive unsolicited notifications from the server you must handle these, even if you don't need them (**hint: you need them**).

```go
notifs := c.Notifications()
go func() {
    for n := range notifs {
        // it's possible the pool sends more than just jobs
        // but that's the main thing
        parseJob(n)
    }
}()
```

Use the client to call the remote service.  One difference from the standard library rpc.Client is that calls have a timeout (30s by default).  This can be adjusted by setting `stratum.CallTimeout`.

```go
// I recommend not using map[string]interface{}
reply := make(map[string]interface{})

// Call will block until finished
err := c.Call("login", params, &reply)
if err != nil {
    // handle it!
}
```

JSON RPC 2.0 allows the client to send Notifications to the server without expecting a reply.  I don't know of a common use case for this with Monero mining strata, but the functionality is there if you need it!

`err := c.Notify("methodname", params)`

### Server

Create a server and Register a struct that will provide the service methods.  **You must register your service with the name "mining".***

`s := stratum.NewServer()`
`s.RegisterName("mining", &Mining{})`

Mining should be an instance of something like this:

```go
type Mining struct{}

// Auth is special login method for Coinhive miners
func (m *Mining) Auth(p Params, resp Reply) error {
    return nil
}

func (m *Mining) Login(p Params, resp Reply) error {
    return nil
}

func (m *Mining) Getjob(p Params, resp Reply) error {
    return nil
}

func (m *Mining) Submit(p Params, resp Reply) error {
    return nil
}

func (m *Mining) Keepalived(p Params, resp Reply) error {
    return nil
}
```

Again, [you can check out a mostly complete implementation in this project.](https://github.com/trey-jones/xmrwasp)

#### Codecs

There are currently 2 server codecs included here.  `DefaultServerCodec` will handle typical Monero stratum+tcp connections.

`CoinhiveServerCodec` will handle what I will refer to as "coinhive-style" connections typical to browser miners.  One FOSS variant is [CryptoNoter](https://github.com/cryptonoter/CryptoNoter).  **Note well that the connection used to serve the codec must be an io.ReadWriteCloser.** One library that exposes such a connection for websockets is [this one.](https://github.com/eyesore/ws)  Full Disclosure: I am the author.

Create the appropriate codec for your application:

`codec := stratum.NewDefaultServerCodecContext(ctx, conn)`

OR

`codec := stratum.NewCoinhiveServerCodecContext(ctx, conn)`

Serve it up, usually in a separate goroutine:

`go s.ServeCodec(codec)`

## Support this Project

If you find the project useful, consider helping me out.  If crypto keeps it up, maybe it will help pay for my daughters' weddings or education, or something.

* Monero: `47sfBPDL9qbDuF5stdrE8C6gVQXf15SeTN4BNxBZ3Ahs6LTayo2Kia2YES7MeN5FU7MKUrWAYPhmeFUYQ3v4JBAvKSPjigi`
* Bitcoin: `1NwemnZSLhJLnNUbzXvER6yNX55pv9tAcv`

I will also review any issues or pull requests that come through here.


