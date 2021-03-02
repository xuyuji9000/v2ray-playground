# Directory Structure

```
./
├── app
│   ├── app.go
│   ├── commander
│   │   ├── commander.go
│   │   ├── config.pb.go
│   │   ├── config.proto
│   │   ├── errors.generated.go
│   │   ├── outbound.go
│   │   └── service.go
│   ├── dispatcher
│   │   ├── config.pb.go
│   │   ├── config.proto
│   │   ├── default.go
│   │   ├── dispatcher.go
│   │   ├── errors.generated.go
│   │   ├── sniffer.go
│   │   ├── stats.go
│   │   └── stats_test.go
│   ├── proxyman
│   │   ├── inbound
│   │   └── outbound
├── features
│   ├── inbound
│   │   └── inbound.go
│   ├── outbound
│   │   └── outbound.go
├── infra
│   └── conf
├── main
│   ├── json
│   │   └── config_json.go
│   └── main.go
├── proxy
│   ├── http
│   │   ├── client.go
│   │   ├── config.go
│   │   ├── config.pb.go
│   │   ├── config.proto
│   │   ├── errors.generated.go
│   │   ├── http.go
│   │   └── server.go
│   ├── proxy.go
│   └── vmess
├── transport
└── v2ray.go
```

# Software Initiation 

``` go
// ./main/main.go

import (
    // ...
    // This line [initiate][1]
	_ "v2ray.com/core/main/distro/all"
)

// ...


func main() {

    // ...
    // Get software instance 
    server, err := startV2Ray()
    // ...

    // Start software instance 
    if err := server.Start(); err != nil {
		fmt.Println("Failed to start", err)
		os.Exit(-1)
    }
    // ...
}


func startV2Ray() {
    // ...
    // Load software configuration into memory
    config, err := core.LoadConfig(GetConfigFormat(), configFiles[0], configFiles) 
    // ...

    // Prepare software instance based on configuration
    server, err := core.New(config)
    // ...
}

```


> [What does an underscore in front of an import statement mean?](https://stackoverflow.com/questions/21220077/what-does-an-underscore-in-front-of-an-import-statement-mean)


## Load software configuration into memory 

More logic related to loading from json configuration file.

``` go
// config.go
func LoadConfig(formatName string, filename string, input interface{}) (*Config, error) {
    // ...
    if f, found := configLoaderByName[formatName]; found {
		return f.Loader(input)
    }
    // ...
}
```

``` go
// ./main/json/config_json.go
func init() {
    // RegisterConfigLoader 
    // Register load function at the software initiation 
    // This pattern used by more object creations inside the software.
	common.Must(core.RegisterConfigLoader(&core.ConfigFormat{
		Name:      "JSON",
		Extension: []string{"json"},
		Loader: func(input interface{}) (*core.Config, error) {
			switch v := input.(type) {
			case cmdarg.Arg:
				r, err := confloader.LoadExtConfig(v)
				if err != nil {
					return nil, newError("failed to execute v2ctl to convert config file.").Base(err).AtWarning()
				}
				return core.LoadConfig("protobuf", "", r)
			case io.Reader:
				return serial.LoadJSONConfig(v)
			default:
				return nil, newError("unknow type")
			}
		},
	}))
}

```

``` go
// ./infra/conf/serial/loader.go


// ...
func LoadJSONConfig(reader io.Reader) (*core.Config, error) {
	jsonConfig, err := DecodeJSONConfig(reader)
	if err != nil {
		return nil, err
	}

    // Convert json format configuration into protobuf format
	pbConfig, err := jsonConfig.Build()
	if err != nil {
		return nil, newError("failed to parse json config").Base(err)
	}

	return pbConfig, nil
}
```

``` go
// ./infra/conf/v2ray.go

// Config data structure
type Config struct {
	Port            uint16                 `json:"port"` // Port of this Point server. Deprecated.
	LogConfig       *LogConfig             `json:"log"`
	RouterConfig    *RouterConfig          `json:"routing"`
	DNSConfig       *DnsConfig             `json:"dns"`
	InboundConfigs  []InboundDetourConfig  `json:"inbounds"`
	OutboundConfigs []OutboundDetourConfig `json:"outbounds"`
	InboundConfig   *InboundDetourConfig   `json:"inbound"`        // Deprecated.
	OutboundConfig  *OutboundDetourConfig  `json:"outbound"`       // Deprecated.
	InboundDetours  []InboundDetourConfig  `json:"inboundDetour"`  // Deprecated.
	OutboundDetours []OutboundDetourConfig `json:"outboundDetour"` // Deprecated.
	Transport       *TransportConfig       `json:"transport"`
	Policy          *PolicyConfig          `json:"policy"`
	Api             *ApiConfig             `json:"api"`
	Stats           *StatsConfig           `json:"stats"`
	Reverse         *ReverseConfig         `json:"reverse"`
}

// Convert into protobuf format configuration
func (c *Config) Build() (*core.Config, error) {
    // ...
}
```

## Prepare software instance based on configuration

``` go
// ./v2ray.go

// ...
func New(config *Config) (*Instance, error) {
	var server = &Instance{ctx: context.Background()}

	err, done := initInstanceWithConfig(config, server)
	if done {
		return nil, err
	}

	return server, nil
}
// ...


// Initiate software insance base on configuration
func initInstanceWithConfig(config *Config, server *Instance) (error, bool) {
    // ...
}
// ...


```

## Start instance 

``` go
// ./v2ray.go

// ...
func (s *Instance) Start() error {
	s.access.Lock()
	defer s.access.Unlock()

	s.running = true
	for _, f := range s.features {
		if err := f.Start(); err != nil {
			return err
		}
	}

	newError("V2Ray ", Version(), " started").AtWarning().WriteToLog()

	return nil
}

```


# Dataflow

## Local to Inbound 

## Intorduce "inbound->dispatch->outbound" dataflow

### Inbound handler initiation

``` go
// ./app/proxyman/inbound/inbound.go

// ...
func NewHandler(ctx context.Context, config *core.InboundHandlerConfig) (inbound.Handler, error) {
	// ...
	allocStrategy := receiverSettings.AllocationStrategy
	if allocStrategy == nil || allocStrategy.Type == proxyman.AllocationStrategy_Always {
		return NewAlwaysOnInboundHandler(ctx, tag, receiverSettings, proxySettings)
	}
	// ...
}

```

``` go
// ./app/proxyman/always.go

// ...
// Prepare handler with workers attached
func NewAlwaysOnInboundHandler(ctx context.Context, tag string, receiverConfig *proxyman.ReceiverConfig, proxyConfig interface{}) (*AlwaysOnInboundHandler, error) {

	// ...
	h := &AlwaysOnInboundHandler{
		proxy: p,
		// Prepare dispatcher
		mux:   mux.NewServer(ctx),
		tag:   tag,
	}

	// ...
	// Attach a worker to each assigned port 
	for port := pr.From; port <= pr.To; port++ {
		if net.HasNetwork(nl, net.Network_TCP) {
			newError("creating stream worker on ", address, ":", port).AtDebug().WriteToLog()

			worker := &tcpWorker{
				address:         address,
				port:            net.Port(port),
				proxy:           p,
				stream:          mss,
				recvOrigDest:    receiverConfig.ReceiveOriginalDestination,
				tag:             tag,

				// This is will be discussed later, dispatch origin
				dispatcher:      h.mux,
				sniffingConfig:  receiverConfig.GetEffectiveSniffingSettings(),
				uplinkCounter:   uplinkCounter,
				downlinkCounter: downlinkCounter,
				ctx:             ctx,
			}
			h.workers = append(h.workers, worker)
		}
		// ...
	}
	// ...
}
// ...

// Start all workers when start handler
func (h *AlwaysOnInboundHandler) Start() error {
	for _, worker := range h.workers {
		if err := worker.Start(); err != nil {
			return err
		}
	}
	return nil
}
// ...
```

``` go
// ./app/proxyman/inbound/worker.go

// ...

func (w *tcpWorker) Start() error {
	ctx := context.Background()

	// Handle connection from listener with callback()
	hub, err := internet.ListenTCP(ctx, w.address, w.port, w.stream, func(conn internet.Connection) {
		go w.callback(conn)
	})
	if err != nil {
		return newError("failed to listen TCP on ", w.port).AtWarning().Base(err)
	}
	w.hub = hub
	return nil
}
// ...

func (w *tcpWorker) callback(conn internet.Connection) {
	// ...
	// Invoke proxy specific logic to Process connection
	if err := w.proxy.Process(ctx, net.Network_TCP, conn, w.dispatcher); err != nil {
		newError("connection ends").Base(err).WriteToLog(session.ExportIDToError(ctx))
	}
	// ...
}
// ...

```

``` go
// ./proxy/http/server.go

// ...
func (s *Server) Process(ctx context.Context, network net.Network, conn internet.Connection, dispatcher routing.Dispatcher) error {
	// ...
	err = s.handlePlainHTTP(ctx, request, conn, dest, dispatcher)
	// ...
}
// ...


// ...
func (s *Server) handlePlainHTTP(ctx context.Context, request *http.Request, writer io.Writer, dest net.Destination, dispatcher routing.Dispatcher) error {
	// ...
	// Get link, which is used to pass data back and forth with outbound handler
	link, err := dispatcher.Dispatch(ctx, dest)
	// ...

	// Write to outbound handler through link.Writer
	requestDone := func() error {
		request.Header.Set("Connection", "close")

		requestWriter := buf.NewBufferedWriter(link.Writer)
		common.Must(requestWriter.SetBuffered(false))
		if err := request.Write(requestWriter); err != nil {
			return newError("failed to write whole request").Base(err).AtWarning()
		}
		return nil
	}

	// Read from outbound handler from link.Reader
	// Write received response to local client
	responseDone := func() error {
		responseReader := bufio.NewReaderSize(&buf.BufferedReader{Reader: link.Reader}, buf.Size)
		response, err := http.ReadResponse(responseReader, request)
		if err == nil {
			http_proto.RemoveHopByHopHeaders(response.Header)
			if response.ContentLength >= 0 {
				response.Header.Set("Proxy-Connection", "keep-alive")
				response.Header.Set("Connection", "keep-alive")
				response.Header.Set("Keep-Alive", "timeout=4")
				response.Close = false
			} else {
				response.Close = true
				result = nil
			}
		} else {
			newError("failed to read response from ", request.Host).Base(err).AtWarning().WriteToLog(session.ExportIDToError(ctx))
			response = &http.Response{
				Status:        "Service Unavailable",
				StatusCode:    503,
				Proto:         "HTTP/1.1",
				ProtoMajor:    1,
				ProtoMinor:    1,
				Header:        http.Header(make(map[string][]string)),
				Body:          nil,
				ContentLength: 0,
				Close:         true,
			}
			response.Header.Set("Connection", "close")
			response.Header.Set("Proxy-Connection", "close")
		}
		if err := response.Write(writer); err != nil {
			return newError("failed to write response").Base(err).AtWarning()
		}
		return nil
	}

	// invoke response logic after request logic
	if err := task.Run(ctx, requestDone, responseDone); err != nil {
		common.Interrupt(link.Reader)
		common.Interrupt(link.Writer)
		return newError("connection ends").Base(err)
	}
}
// ...
```

### Dispatch 

#### Inbound dispatch

``` go
// ./common/mux/server.go
type Server struct {
	dispatcher routing.Dispatcher
}

// ...

// NewServer creates a new mux.Server.
// Get dispatch instance stored in context
func NewServer(ctx context.Context) *Server {
	s := &Server{}
	core.RequireFeatures(ctx, func(d routing.Dispatcher) {
		s.dispatcher = d
	})
	return s
}

// ...

// Dispatch impliments routing.Dispatcher
func (s *Server) Dispatch(ctx context.Context, dest net.Destination) (*transport.Link, error) {
	if dest.Address != muxCoolAddress {
		return s.dispatcher.Dispatch(ctx, dest)
	}
	// ...
}

// ...
```


``` go
// ./app/dispatcher/default.go


// ...

func init() {
	common.Must(common.RegisterConfig((*Config)(nil), func(ctx context.Context, config interface{}) (interface{}, error) {
		d := new(DefaultDispatcher)
		if err := core.RequireFeatures(ctx, func(om outbound.Manager, router routing.Router, pm policy.Manager, sm stats.Manager) error {
			return d.Init(config.(*Config), om, router, pm, sm)
		}); err != nil {
			return nil, err
		}
		return d, nil
	}))
}

// Init initializes DefaultDispatcher.
func (d *DefaultDispatcher) Init(config *Config, om outbound.Manager, router routing.Router, pm policy.Manager, sm stats.Manager) error {
	d.ohm = om
	d.router = router
	d.policy = pm
	d.stats = sm
	return nil
}
// ...


// Dispatch implements routing.Dispatcher.
func (d *DefaultDispatcher) Dispatch(ctx context.Context, destination net.Destination) (*transport.Link, error) {
	// ...
	// Prepare link(underlying, they are two pipes)
	inbound, outbound := d.getLink(ctx)
	// ...
	if destination.Network != net.Network_TCP || !sniffingRequest.Enabled {
		go d.routedDispatch(ctx, outbound, destination)
	} else {
		go func() {
			cReader := &cachedReader{
				reader: outbound.Reader.(*pipe.Reader),
			}
			outbound.Reader = cReader
			result, err := sniffer(ctx, cReader)
			if err == nil {
				content.Protocol = result.Protocol()
			}
			if err == nil && shouldOverride(result, sniffingRequest.OverrideDestinationForProtocol) {
				domain := result.Domain()
				newError("sniffed domain: ", domain).WriteToLog(session.ExportIDToError(ctx))
				destination.Address = net.ParseAddress(domain)
				ob.Target = destination
			}
			// outbound link used here
			d.routedDispatch(ctx, outbound, destination)
		}()
	}
	return inbound, nil
}

// ...
func (d *DefaultDispatcher) routedDispatch(ctx context.Context, link *transport.Link, destination net.Destination) {
	var handler outbound.Handler
	// ...
	if handler == nil {
		handler = d.ohm.GetDefaultHandler()
	}
	// ...
	// Use default outbound handler
	handler.Dispatch(ctx, link)
}
// ...


func (d *DefaultDispatcher) getLink(ctx context.Context) (*transport.Link, *transport.Link) {
	opt := pipe.OptionsFromContext(ctx)
	uplinkReader, uplinkWriter := pipe.New(opt...)
	downlinkReader, downlinkWriter := pipe.New(opt...)

	inboundLink := &transport.Link{
		Reader: downlinkReader,
		Writer: uplinkWriter,
	}

	outboundLink := &transport.Link{
		Reader: uplinkReader,
		Writer: downlinkWriter,
	}

	// ...

	return inboundLink, outboundLink
}

// ...

```


#### Outbound dispatch

``` go
// ./app/proxyman/outbound/handler.go
// Dispatch implements proxy.Outbound.Dispatch.
func (h *Handler) Dispatch(ctx context.Context, link *transport.Link) {
	if h.mux != nil && (h.mux.Enabled || session.MuxPreferedFromContext(ctx)) {
		if err := h.mux.Dispatch(ctx, link); err != nil {
			newError("failed to process mux outbound traffic").Base(err).WriteToLog(session.ExportIDToError(ctx))
			common.Interrupt(link.Writer)
		}
	} else {
		if err := h.proxy.Process(ctx, link, h); err != nil {
			// Ensure outbound ray is properly closed.
			newError("failed to process outbound traffic").Base(err).WriteToLog(session.ExportIDToError(ctx))
			common.Interrupt(link.Writer)
		} else {
			common.Must(common.Close(link.Writer))
		}
		common.Interrupt(link.Reader)
	}
}
```

``` go
// ./proxy/vmess/outbound/outbound.go

// ...
func (v *Handler) Process(ctx context.Context, link *transport.Link, dialer internet.Dialer) error {
	// ...
	input := link.Reader
	output := link.Writer
	// ...


	requestDone := func() error {
		defer timer.SetTimeout(sessionPolicy.Timeouts.DownlinkOnly)

		writer := buf.NewBufferedWriter(buf.NewWriter(conn))
		if err := session.EncodeRequestHeader(request, writer); err != nil {
			return newError("failed to encode request").Base(err).AtWarning()
		}

		bodyWriter := session.EncodeRequestBody(request, writer)

		// Write request to connection
		if err := buf.CopyOnceTimeout(input, bodyWriter, time.Millisecond*100); err != nil && err != buf.ErrNotTimeoutReader && err != buf.ErrReadTimeout {
			return newError("failed to write first payload").Base(err)
		}

		if err := writer.SetBuffered(false); err != nil {
			return err
		}

		if err := buf.Copy(input, bodyWriter, buf.UpdateActivity(timer)); err != nil {
			return err
		}

		if request.Option.Has(protocol.RequestOptionChunkStream) {
			if err := bodyWriter.WriteMultiBuffer(buf.MultiBuffer{}); err != nil {
				return err
			}
		}

		return nil
	}

	responseDone := func() error {
		defer timer.SetTimeout(sessionPolicy.Timeouts.UplinkOnly)

		reader := &buf.BufferedReader{Reader: buf.NewReader(conn)}
		header, err := session.DecodeResponseHeader(reader)
		if err != nil {
			return newError("failed to read header").Base(err)
		}
		v.handleCommand(rec.Destination(), header.Command)

		bodyReader := session.DecodeResponseBody(request, reader)

		// Put data back through dispatch pipe
		return buf.Copy(bodyReader, output, buf.UpdateActivity(timer))
	}


	var responseDonePost = task.OnSuccess(responseDone, task.Close(output))
	if err := task.Run(ctx, requestDone, responseDonePost); err != nil {
		return newError("connection ends").Base(err)
	}

	// ...
}
// ...
```




2. Introduce "outbound->Internet->inbound" dataflow





## Outbound through Internet to Inbound 

### Register Dialer

``` go
// ./transport/internet/dialer.go

// ...

// RegisterTransportDialer registers a Dialer with given name.
func RegisterTransportDialer(protocol string, dialer dialFunc) error {
	if _, found := transportDialerCache[protocol]; found {
		return newError(protocol, " dialer already registered").AtError()
	}
	transportDialerCache[protocol] = dialer
	return nil
}
// ...
```


``` go

// ./transport/internet/tcp/dialer.go

// Dial dials a new TCP connection to the given destination.
func Dial(ctx context.Context, dest net.Destination, streamSettings *internet.MemoryStreamConfig) (internet.Connection, error) {
	newError("dialing TCP to ", dest).WriteToLog(session.ExportIDToError(ctx))
	conn, err := internet.DialSystem(ctx, dest, streamSettings.SocketSettings)
	if err != nil {
		return nil, err
	}

	if config := tls.ConfigFromStreamSettings(streamSettings); config != nil {
		tlsConfig := config.GetTLSConfig(tls.WithDestination(dest))
		if config.IsExperiment8357() {
			conn = tls.UClient(conn, tlsConfig)
		} else {
			conn = tls.Client(conn, tlsConfig)
		}
	}

	tcpSettings := streamSettings.ProtocolSettings.(*Config)
	if tcpSettings.HeaderSettings != nil {
		headerConfig, err := tcpSettings.HeaderSettings.GetInstance()
		if err != nil {
			return nil, newError("failed to get header settings").Base(err).AtError()
		}
		auth, err := internet.CreateConnectionAuthenticator(headerConfig)
		if err != nil {
			return nil, newError("failed to create header authenticator").Base(err).AtError()
		}
		conn = auth.Client(conn)
	}
	return internet.Connection(conn), nil
}

func init() {
	common.Must(internet.RegisterTransportDialer(protocolName, Dial))
}
```

``` go

// ./transport/internet/system_dialer.go

func (d *DefaultSystemDialer) Dial(ctx context.Context, src net.Address, dest net.Destination, sockopt *SocketConfig) (net.Conn, error) {
	// ...

	dialer := &net.Dialer{
		Timeout:   time.Second * 16,
		DualStack: true,
		LocalAddr: resolveSrcAddr(dest.Network, src),
	}

	// ...
	return dialer.DialContext(ctx, dest.Network.SystemString(), dest.NetAddr())
}
```

### Get outbound connection

``` go
// ./proxy/vmess/outbound/outbound.go

func (v *Handler) Process(ctx context.Context, link *transport.Link, dialer internet.Dialer) error {
	//...
	var conn internet.Connection


	err := retry.ExponentialBackoff(5, 200).On(func() error {
		rec = v.serverPicker.PickServer()
		// get conection through dialing remote address
		rawConn, err := dialer.Dial(ctx, rec.Destination())
		if err != nil {
			return err
		}
		conn = rawConn

		return nil
	})

	// ...
	// write to remote connection
	requestDone := func() error {
		// ...

		writer := buf.NewBufferedWriter(buf.NewWriter(conn))
		
		// ...

		bodyWriter := session.EncodeRequestBody(request, writer)
		if err := buf.CopyOnceTimeout(input, bodyWriter, time.Millisecond*100); err != nil && err != buf.ErrNotTimeoutReader && err != buf.ErrReadTimeout {
			return newError("failed to write first payload").Base(err)
		}

		if err := writer.SetBuffered(false); err != nil {
			return err
		}

		if err := buf.Copy(input, bodyWriter, buf.UpdateActivity(timer)); err != nil {
			return err
		}

		if request.Option.Has(protocol.RequestOptionChunkStream) {
			if err := bodyWriter.WriteMultiBuffer(buf.MultiBuffer{}); err != nil {
				return err
			}
		}

		return nil
	}

	// Read from remote connection
	responseDone := func() error {
		// ...

		reader := &buf.BufferedReader{Reader: buf.NewReader(conn)}
		header, err := session.DecodeResponseHeader(reader)
		if err != nil {
			return newError("failed to read header").Base(err)
		}
		v.handleCommand(rec.Destination(), header.Command)

		bodyReader := session.DecodeResponseBody(request, reader)

		return buf.Copy(bodyReader, output, buf.UpdateActivity(timer))
	}
}

```

``` go
// ./app/proxyman/outbound/handler.go
func (h *Handler) Dial(ctx context.Context, dest net.Destination) (internet.Connection, error) {
	// ...
	conn, err := internet.Dial(ctx, dest, h.streamSettings)
	return h.getStatCouterConnection(conn), err
}
```

``` go
// ./transport/internet/dialer.go

// ...
func Dial(ctx context.Context, dest net.Destination, streamSettings *MemoryStreamConfig) (Connection, error) {
	if dest.Network == net.Network_TCP {
		if streamSettings == nil {
			s, err := ToMemoryStreamConfig(nil)
			if err != nil {
				return nil, newError("failed to create default stream settings").Base(err)
			}
			streamSettings = s
		}

		protocol := streamSettings.ProtocolName
		dialer := transportDialerCache[protocol]
		if dialer == nil {
			return nil, newError(protocol, " dialer not registered").AtError()
		}
		return dialer(ctx, dest, streamSettings)
	}
	// ...
}
```

# Reference 