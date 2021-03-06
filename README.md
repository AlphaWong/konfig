[![Build Status](https://travis-ci.org/lalamove/konfig.svg?branch=master)](https://travis-ci.org/lalamove/konfig)
[![codecov](https://codecov.io/gh/lalamove/konfig/branch/master/graph/badge.svg)](https://codecov.io/gh/lalamove/konfig)
[![Go Report Card](https://goreportcard.com/badge/github.com/lalamove/konfig)](https://goreportcard.com/report/github.com/lalamove/konfig)
[![Go doc](http://img.shields.io/badge/go-documentation-blue.svg?style=flat-square
)](https://godoc.org/github.com/lalamove/konfig)

# Konfig
Composable, observable and performant config management for Go. 
It allows you to compose configurations from multiple sources with reload mechanism making it simple to build stateful apps. 

# Why another config package?
Most config packages for Golang are hardly extensible and composable and make it complex to build apps which can reload their state dynamically. Also, few of them come with sources such as Vault, Etcd and multiple encoding formats. 

konfig is built around 4 small interfaces: 
- Loader
- Watcher 
- Parser 
- Closer  

Konfig features include: 
- **Dynamic** configuration loading
- **Composable** load configs from multiple sources. At Lalamove we load configs from Vault, files and etcd. 
- **Polyglot** load configs from multiple format. Konfig support JSON, YAML, TOML, Key=Value. 
- **Fast, Lockfree, Thread safe Read** Reads are **up to 10x faster than Viper**. 
- **Observable config, Hot Reload** mechanism and tooling to manage state. 
- **Typed Read** get typed values from config or bind a struct. 
- **Metrics** exposed prometheus metrics telling you how many times a config is reloaded, if it failed, and how long it takes to reload!

# Get started 
```sh
go get github.com/lalamove/konfig
```

Load and watch a json formatted config file. 
```go
var configFiles = []klfile.File{
	{
		Path: "./config.json",
		Parser: json.Parser,
	},
}

func init() {
	konfig.Init(konfig.DefaultConfig())
}

func main() {
	// load from json file	
	konfig.RegisterLoaderWatcher(
		klfile.New(&klfile.Config{
			Files: 			configFiles,
			Watch: 			true,
		}),
		// optionally you can pass config hooks to run when a file is changed
		func(c Store) error {
			return nil	
		},
	)

	if err := konfig.LoadWatch(); err != nil {
		log.Fatal(err)
	}

	// retrieve value from config file
	konfig.Bool("debug")
}
```

# Store
The Store is the base of the config package. It holds and gives access to values stored by keys.

## Creating a Store
You can create a global Store by calling `konfig.Init(*konfig.Config)`: 
```go
konfig.Init(konfig.DefaultConfig())
```
The global store is accessible directly from the package:
```go
konfig.Get("foo") // calls store.Get("foo")
```

You can create a new store by calling `konfig.New(*konfig.Config)`:
```go
s := konfig.New(konfig.DefaultConfig())
```

## Loading and Watching a Store
After registering Loaders and Watchers in the `konfig.Store`, you must load and watch the store. 

You can do both by calling `LoadWatch`:
```go
if err := konfig.LoadWatch(); err != nil {
    log.Fatal(err)
}
```

You can call `Load` only, it will load all loaders and return:
```go
if err := konfig.Load(); err != nil {
    log.Fatal(err)
}
```

And finally you can call `Watch` only, it will start all watchers and return: 
```go
if err := konfig.Watch(); err != nil {
    log.Fatal(err)
}
```


# Loaders
Loaders load config values into the store. A loader is an implementation of the loader interface. 
```go
type Loader interface {
	// Loads the config and add it to the Store
	Load(Store) error
	// MaxRetry returns the maximum number of times to allow retrying on load failure
	MaxRetry() int
	// RetryDelay returns the delay to wait before retrying
	RetryDelay() time.Duration
}
```
You can register loaders in the config individually or with a watcher.

### Register a loader by itself:
```go
configLoader := konfig.RegisterLoader(
	klfile.New(
 		&klfile.Config{
			Files: []klfile.File{
				{
					Parser: kpjson.Parser,
					Path: "./konfig.json",
				},
			},
		},
	),
)
```

### Register a loader with a watcher:  
To register a loader and a watcher together, you must register a `LoaderWatcher` which is an interface that implements both the `Loader` and the `Watcher` interface.
```go
configLoader := konfig.RegisterLoaderWatcher(
	klfile.New(
 		&klfile.Config{
			Files: []klfile.File{
				{
					Parser: kpjson.Parser,
					Path: "./konfig.json",
				},
			},
			Watch: true,
		},
	),
)
```
You can also compose a loader and a watcher to create a `LoaderWatcher`: 
```go
configLoader := konfig.RegisterLoaderWatcher(
	// it creates a LoaderWatcher from a loader and a watcher
	konfig.NewLoaderWatcher(
		someLoader,
		someWatcher,
	),
)
```
### Built in loaders
Config already has the following loaders, they all have a built in watcher:
- [File Loader](loader/klfile/README.md)

Loads configs from files which can be watched. Files can have different parsers to load different formats. It has a built in file watcher which triggers a config reload (running hooks) when files are modified. 

- [Vault Loader](loader/klvault/README.md)

Loads configs from vault secrets. It has a built in Poll Watcher which triggers a config reload (running hooks) before the secret and the token from the auth provider expires.

- [HTTP Loader](loader/klhttp/README.md)

Loads configs from HTTP sources. Sources can have different parsers to load different formats. It has a built in Poll Diff Watcher which triggers a config reload (running hooks) if data is different.  

- [Etcd Loader](loader/kletcd/README.md)

Loads configs from Etcd keys. Keys can have different parser to load different formats. It has a build in Poll Diff Watcher which triggers a config reload (running hooks) if data is different.

- [ENV Loader](loader/klenv/README.md)

Loads configs from environment variables.

- [Flag Loader](loader/klflag/README.md)

Loads configs from command line flags.


### Parsers
Parsers parse an `io.Reader` into a `konfig.Store`. These are used by some loaders to parse the data they fetch into the config store. the File Loader, Etcd Loader and HTTP Loader use Parsers. 

Config already has the following parsers: 
- [JSON Parser](parser/kpjson/README.md)
- [TOML Parser](parser/kptoml/README.md)
- [YAML Parser](parser/kpyaml/README.md)
- [KV Parser](parser/kpkeyval/README.md) 
- [Map Parser](parser/kpmap/README.md)

# Watchers
Watchers trigger a call on a Loader on events. A watcher is an implementation of the `Watcher` interface.
```go
type Watcher interface {
	// Start starts the watcher, it must not be blocking.
	Start() error
	// Done indicate wether the watcher is done or not
	Done() <-chan struct{}
	// Watch should block until an event unlocks it
	Watch() <-chan struct{}
	// Close closes the watcher, it returns a non nil error if it is already closed
	// or something prevents it from closing properly.
	Close() error
}
```
### Built in watchers
Config already has the following watchers: 
- [File Watcher](watcher/filewatcher/README.md)

Watches files for changes. 

- [Poll Watcher](watcher/filewatcher/README.md)

Sends events at a given rate, or if diff is enabled. It takes a Getter and fetches the data
at a given rate. If data is different, it sends an event. 

# Hooks
Hooks are functions ran after a successful loader Load call. They are used to reload the state of the application on a config change.

### Registering a loader with some hooks
You can register a loader or a loader watcher with hooks.
```go
configLoader := konfig.RegisterLoaderWatcher(
	klfile.New(
 		&klfile.Config{
			Files: []klfile.File{
				{
					Parser: yamlparser.Parser,
					Path: "./konfig.yaml",
				},
			},
			Watch: true,
		},
	),
	func(s konfig.Store) error {
		// Here you should reload the state of your app
		return nil
	},
)
```

### Adding hooks to an existing loader
You can register a loader or a loader watcher with hooks.
```go
configLoader.AddHooks(
	func(s konfig.Store) error {
		// Here you should reload the state of your app	
		return nil
	},
	func(s konfig.Store) error {
		// Here you should reload the state of your app	
		return nil
	},
)
```

# Closers
Closers can be added to the config so that if config fails to load it will close the registered Closers.
```go
type Closer interface {
	Close() error
}
``` 
## Register a Closer
```go
konfig.RegisterCloser(closer)
```

# Config groups
You can namespace your configs using config Groups. 
```go
konfig.Group("db").RegisterLoaderWatcher(	
	klfile.New(
 		&klfile.Config{
			Files: []klfile.File{
				{
					Parser: yamlparser.Parser,
					Path: "./db.yaml",
				},
			},
			Watch: true,
		},
	),
)

// accessing grouped config 
dbHost := konfig.Group("db").MustString("credentials.host")
```

# Binding a type to a Store
You can bind a type to the konfig store if you want your config values to be unmarshaled to a **struct** or a **map[string]interface{}**. Then you can access an instance of that type in a thread safe manner (in order to be safe for dynamic config updates).

Let's see with an example of a json config file: 
```json
{
    "addr": ":8080",
    "debug": true,
    "db": {
        "username": "foo"
    },
    "redis": {
        "host": "127.0.0.1"
    }
}
```

```go
type DBConfig struct {
    Username string
}
type Config struct {
    Addr        string
    Debug       string
    DB          DBConfig `konfig:"db"`
    RedisHost   string `konfig:"redis.host"`
}

// we init the root konfig store
konfig.Init(konfig.DefaultConfig())

// we bind the Config struct to the konfig.Store
konfig.Bind(Config{})

// we register our config file
konfig.RegisterLoaderWatcher(
	klfile.New(
 		&klfile.Config{
			Files: []klfile.File{
				{
					Parser: kpjson.Parser,
					Path: "./config.json",
				},
			},
			Watch: true,
		},
	),
)

// we load our config and start watching
if err := konfig.LoadWatch(); err != nil {
    log.Fatal(err)
}

// Get our config value
c := konfig.Value().(Config)

fmt.Println(c.Addr) // :8080
```

Note that you can compose your config sources. For example, have your credentials come from Vault and be renewed often and have the rest of your config loaded from a file and be updated on file change. 

**It is important to understand how Konfig unmarshals your config values into your struct.**
When a Loader calls *konfig.Set()*, if the konfig store has a value bound to it, it will try to unmarshal the key to the bound value. 
- First, it will look for field tags in the struct, if a tag matches exactly the key, it will unmarshal the key to the struct field.
- Then, it will do a EqualFold on the field name and the key, if they match, it will unmarshal the key to the struct field.
- Then, if the key has a dot, it will check if the tag or the field name (to lowercase) is a prefix of the key, if yes, it will check if the type of the field is a struct of pointer, if yes, it will check the struct using whats after the prefix as the key. 


# Read from config
Apart from reading from the bound config value, konfig provides several methods to read values.

Every method to retrieve config values come in 2 flavours:
- **Get** reads a the value at the given key. If key is not present it returns the zero value of the type.
- **MustGet**  reads a the value at the given key. If key is not present it panics.

All methods to read values from a Store:
```go
// Exists checks wether the key k is set in the store.
Exists(k string) bool

// Get gets the value with the key k fron the store. If the key is not set, Get returns nil. To check wether a value is really set, use Exists.
Get(k string) interface{}
// MustGet tries to get the value with the key k from the store. If the key k does not exist in the store, MustGet panics.
MustGet(k string) interface{}

// MustString tries to get the value with the key k from the store and casts it to a string. If the key k does not exist in the store, MustGet panics.
MustString(k string) string
// String tries to get the value with the key k from the store and casts it to a string. If the key k does not exist it returns the Zero value.
String(k string) string

// MustInt tries to get the value with the key k from the store and casts it to a int. If the key k does not exist in the store, MustInt panics.
MustInt(k string) int
// Int tries to get the value with the key k from the store and casts it to a int. If the key k does not exist it returns the Zero value.
Int(k string) int

// MustFloat tries to get the value with the key k from the store and casts it to a float. If the key k does not exist in the store, MustFloat panics.
MustFloat(k string) float64
// Float tries to get the value with the key k from the store and casts it to a float. If the key k does not exist it returns the Zero value.
Float(k string) float64

// MustBool tries to get the value with the key k from the store and casts it to a bool. If the key k does not exist in the store, MustBool panics.
MustBool(k string) bool
// Bool tries to get the value with the key k from the store and casts it to a bool. If the key k does not exist it returns the Zero value.
Bool(k string) bool

// MustDuration tries to get the value with the key k from the store and casts it to a time.Duration. If the key k does not exist in the store, MustDuration panics.
MustDuration(k string) time.Duration
// Duration tries to get the value with the key k from the store and casts it to a time.Duration. If the key k does not exist it returns the Zero value.
Duration(k string) time.Duration

// MustTime tries to get the value with the key k from the store and casts it to a time.Time. If the key k does not exist in the store, MustTime panics.
MustTime(k string) time.Time
// Time tries to get the value with the key k from the store and casts it to a time.Time. If the key k does not exist it returns the Zero value.
Time(k string) time.Time

// MustStringSlice tries to get the value with the key k from the store and casts it to a []string. If the key k does not exist in the store, MustStringSlice panics.
MustStringSlice(k string) []string
// StringSlice tries to get the value with the key k from the store and casts it to a []string. If the key k does not exist it returns the Zero value.
StringSlice(k string) []string

// MustIntSlice tries to get the value with the key k from the store and casts it to a []int. If the key k does not exist in the store, MustIntSlice panics.
MustIntSlice(k string) []int
// IntSlice tries to get the value with the key k from the store and casts it to a []int. If the key k does not exist it returns the Zero value.
IntSlice(k string) []int

// MustStringMap tries to get the value with the key k from the store and casts it to a map[string]interface{}. If the key k does not exist in the store, MustStringMap panics.
MustStringMap(k string) map[string]interface{}
// StringMap tries to get the value with the key k from the store and casts it to a map[string]interface{}. If the key k does not exist it returns the Zero value.
StringMap(k string) map[string]interface{}

// MustStringMapString tries to get the value with the key k from the store and casts it to a map[string]string. If the key k does not exist in the store, MustStringMapString panics.
MustStringMapString(k string) map[string]string
// StringMapString tries to get the value with the key k from the store and casts it to a map[string]string. If the key k does not exist it returns the Zero value.
StringMapString(k string) map[string]string 
```

# Getter
To easily build services which can use dynamically loaded configs you can create getters for specific keys. A getter implements `ngetter.GetterTyped` from [nui](github.com/lalamove/nui) package. It is useful when building stateful apps.

Example with a config value set for the debug key: 
```go
debug := konfig.Getter("debug")

debug.Bool() // true
```

# Metrics
Konfig comes with prometheus metrics.

Two metrics are exposed: 
- Config reloads counter vector with labels 
- Config reload duration summary vector with labels

Example of metrics: 
```
# HELP konfig_loader_reload Number of config loader reload
# TYPE konfig_loader_reload counter
konfig_loader_reload{loader="config-files",result="failure",store="root"} 0.0
konfig_loader_reload{loader="config-files",result="success",store="root"} 1.0

# HELP konfig_loader_reload_duration Histogram for the config reload duration
# TYPE konfig_loader_reload_duration summary
konfig_loader_reload_duration{loader="config-files",store="root",quantile="0.5"} 0.001227641
konfig_loader_reload_duration{loader="config-files",store="root",quantile="0.9"} 0.001227641
konfig_loader_reload_duration{loader="config-files",store="root",quantile="0.99"} 0.001227641
konfig_loader_reload_duration_sum{loader="config-files",store=""} 0.001227641
konfig_loader_reload_duration_count{loader="config-files",store=""} 1.0
```

To enable metrics, you must pass a custom config when creating a config store: 
```go
konfig.Init(&konfig.Config{
    Metrics: true,
    Name: "root",
})
```

# Benchmark
Benchmarks are run on both Viper and Konfig. Benchmark are done on reading ops and show that Konfig is 0 allocs on read and at leat 3x fastet than Viper:
```
goos: linux
goarch: amd64
pkg: github.com/lalamove/konfig/benchmarks
BenchmarkGetKonfig-4            100000000               18.1 ns/op             0 B/op          0 allocs/op
BenchmarkStringKonfig-4         10000000               148 ns/op               0 B/op          0 allocs/op
BenchmarkGetViper-4              5000000               347 ns/op              32 B/op          2 allocs/op
BenchmarkStringViper-4           3000000               429 ns/op              32 B/op          2 allocs/op
PASS
ok      github.com/lalamove/konfig/benchmarks   8.225s
```


# Contributing 

Contributions are welcome. To make contributions, fork the repository, create a branch and submit a Pull Request to the master branch. 
