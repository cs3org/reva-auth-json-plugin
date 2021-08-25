# reva-auth-json-plugin
A sample plugin compatible with Reva which allows users to be authenticated from a json database.

# Developing a Reva Plugin

[Reva](https://github.com/cs3org/reva) plugins are developed using the [Go language](https://golang.org/). 

A Reva driver plugin is just a Go package that provides an implementation of the interface exposed by the Reva services. 

Reva plugins are implemented using the [hashicorp go-plugin](https://github.com/hashicorp/go-plugin), which is a plugin system over RPC. The core implementation of the go-plugin framework comes with Reva out of the box, hence you only need to take care of developing the plugins.

## Usage/Configuration

For a plugin to be active for a given Reva instance, it must be declared in the `TOML` configuration file. 

Reva provides users 3 options to configure plugins:

1. **Loading a pre-compiled binary**: To load a pre-compiled plugin binary, just mention the absolute path of the binary. Here is an example of configuring Reva to load `auth-json` plugin:

```toml
[grpc.services.authprovider]
auth_manager = "/path/json/plugin"

[grpc.services.authprovider.auth_managers.reva-auth-json-plugin]
users = "user.demo.json"
```

2. **Compiling and Loading Plugin**: Reva can compile your plugin source code into an executable and load it on to the core. In order to configure Reva to compile the plugin, mention the absolute path to the source code package.

**Note: Reva compiles stores the compiled binary at `/var/tmp/reva/bin/<plugin_type>/<plugin_name>`, wherein plugin_type is the service that plugin belongs to and plugin_name is the name of your plugin.**

```toml
[grpc.services.authprovider]
auth_manager = "/path/to/source_code"

[grpc.services.authprovider.auth_managers.reva-auth-json-plugin]
users = "user.demo.json"
```

3. **Github Repository**: In order to configure Reva to fetch plugins from github, provide the link to the source code repo in the driver configuration. 

```toml
[grpc.services.authprovider]
auth_manager = "github.com/cs3org/reva-auth-json-plugin"

[grpc.services.authprovider.auth_managers.reva-auth-json-plugin]
users = "user.demo.json"
```

Reva would download the source code in `/var/tmp/reva/ext/<plugin_type>/<plugin_name>`, compile the source code and then load the binary onto the core.

## Developing a Plugin

There are a certain steps to be followed in order to develop a reva plugin.

The components that can be created and used in a Reva Plugin are service drivers. These drivers can belong to any of the Reva service, eg: Userprovider, Storageprovider, Authprovider etc. Each service exposes an interface for the plugin to implement.

All you need to do to create a plugin is:

- Create an implementation of the desired interface: A Plugin should implement the interface exposed by the corresponding service. Eg: Userprovider interface.
- Serve the plugin using the [go-plugin's](https://github.com/hashicorp/go-plugin/) plugin.Serve method.

The core handles all of the communication details and go-plugin implementations inside the server.

Your plugin must use the packages from the Reva core to implement the interfaces. You're encouraged to use whatever other packages you want in your plugin implementation. Because plugins are their own processes, there is no danger of colliding dependencies.

- `github.com/cs3org/reva/pkg/<service_type>`: Contains the interface that you have to implement for any give plugin.
- `github.com/hashicorp/go-plugin`: To serve the plugin over RPC. This handles all the inter-process communication.
- `github.com/cs3org/reva/pkg/plugin`: To use the exposed plugin Handshake. 

Basic example of serving your component is shown below. This example consists of a simple `JSON` plugin driver for the [Userprovider](https://github.com/cs3org/reva/blob/master/internal/grpc/services/userprovider/userprovider.go) service. You can find the example code [here](https://github.com/cs3org/reva/blob/master/examples/plugin/json/json.go).

```go
// main.go

import (
   	"github.com/cs3org/reva/pkg/user"
    "github.com/hashicorp/go-plugin"
    revaPlugin "github.com/cs3org/reva/pkg/plugin"
)

// Assume this implements the user.Manager interface
type Manager struct{}

func main() {
    // plugin.Serve serves the implementation over RPC to the core
	plugin.Serve(&plugin.ServeConfig{
		HandshakeConfig: revaPlugin.Handshake,
		Plugins: map[string]plugin.Plugin{
			"userprovider": &user.ProviderPlugin{Impl: &Manager{}},
		},
	})
}
```

The `plugin.Serve` method handles all the details of communicating with Reva core and serving your component over RPC. As long as your struct implements one of the exposed interfaces, Reva will be able to launch your plugin and use it.

The `plugin.Serve` method takes in the plugin configuration, which you would have to define in your plugin source code:

- `HandshakeConfig`: The handshake is defined in `github.com/cs3org/reva/pkg/plugin`:

```go
var Handshake = plugin.HandshakeConfig{
	ProtocolVersion:  1,
	MagicCookieKey:   "BASIC_PLUGIN",
	MagicCookieValue: "reva",
}
```

- `Plugins`:  Plugins is a map which maps the plugin implementation with the plugin name. Currently Reva supports 2 types of plugins (support for more to be added soon):

    - `userprovider`
    - `authprovider`

The implementation should be mapped to the above provided plugin names.