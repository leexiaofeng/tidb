# Proposal: Support Plugin

- Author(s):     [lysu](https://github.com/lysu)
- Last updated:  2018-12-10
- Discussion at:

## Abstract

This proposal proposes to introduce the plugin framework to TiDB to support TiDB plugin development.

## Background

Many cool customized requirements need to be addressed but it is not convenient to merge them into the main TiDB repository. In addition, Go 1.9+ introduces the new plugin support, so we can add a plugin framework to TiDB to make those requirements addressed, and attract more people to build the TiDB ecosystem together.

## Proposal

Add a plugin framework to TiDB.

## Rationale

Adding the plugin framework is based on Go's plugin support, but this framework supports uniform plugin manifest, package, and flexible SPI.

The plugin developer can build a TiDB plugin in 7 steps:

- Choose an exists plugin kind or create new plugin kind if not exists
- Create a normal go package, and add `manifest.toml` like example one.
- Implement the `validate`, `init`, `destroy` methods, which are needed for all plugins.
- Implement the kind special method to implement the plugin logic.
- Use `cmd/pluginpkg` to build plugin binary, and put the plugin binary into the plugin deployment folder.
- Start TiDB with the `-plugin-dir` and `-plugin-load` parameters.
- Run `show plugin` to check it's load status.

## Implementation

### Go Plugin

We build the plugin framework based on Go's plugin support. At first, let's see "what is Go's plugin supported?"

Go's plugin support is simple, just as the document at https://golang.org/pkg/plugin/. We can build and use the plugin in three steps:

- Build the plugin via `go build -buildmode=plugin` in the `main` package to make plugin `.so`.
- Use `plugin.Open` to `dlopen` the plugin's `.so`.
- Use `plugin.Lookup` to `dlsym` to find the symbol in the plugin `.so`.

There is another "undocumented" but important concept: `pluginpath`. Just as previously said we let our plugin code into the `main` package and then `go build -buildmode=plugin` to build a plugin. `pluginpath` is the package path for a plugin after plugin packaged. For example, we have a method named `DoIt` and `pluginpath` be `pkg1`, and then we can use `nm` to see the method name be `pluginpath.DoIt`. 

`pluginpath` can be given by `-ldflags -pluginpath=[path-value]` or generated by [go build](https://github.com/golang/go/blob/3b137dd2df19c261a007b8a620a2182cd679d700/src/cmd/go/internal/work/gc.go#L389)(for 1.11.1, it is the package name if `pluginpath` is built with the package folder or a content hash if built with the source file).

If we load a Go plugin with the same `pluginpath` twice, the second `Load` call will get an error, Go plugin use `pluginpath` to detect duplicate load.

The last thing we need to care about is the Go plugin's dependence. At first, almost all plugins need to depend on TiDB code to do its logic. Go runtime requires that the runtime hash and the link time hash for the dependence package are equal. So we do not need to care about the plugin that depends on some TiDB component. But TiDB code changes, so we need to release a new plugin whenever a TiDB new version is released.

### TiDB Plugin

Go plugin gives us a good start point, but we need to do something more to let the plugin be more uniform and easy to use with TiDB.

#### Manifest

Go Plugin gives us the ability to open a shared library. We need some meta info to self-describe the plugin, and then TiDB can know how to work with the loaded library. The information we need is as follows:

- Plugin name: we need to reload the plugin, so we need to load the same plugin with a different version that is at a much higher level than `pluginpath`.
- Plugin version: plugin version makes it much easier to maintain.
- Simple Dependence check: Go helps to use a check to build the version, but in the real world it is common that a plugin relies on b plugin's some new logic. We try to maintain a simple Required-Version relationship between different plugins.
- Configuration: the plugin is a standalone module, so every plugin will introduce special sysVars just like normal MySQL variables. A user can use those variables to tweak plugin behaviors just like normal MySQL variables do.
- Stats: the plugin will introduce new stats info. TiDB uses Prometheus, so the plugin can easily push metrics to Prometheus.
- Plugin Category and Flexible SPI: TiDB can have limited plugin categories, a new plugin need choose a category and implement SPI defined by those category.

All of above construct the plugin metadata which we usually call `Manifest`. `Manifest` describes the metadata and how others can use the plugin.

We just use the Go plugin mechanism to load the TiDB plugin and the TiDB plugin gives us a `Manifest`. Then just use manifest to interact with the plugin. (Only load/lookup is heavy CGO call, and later call manifest is normal Golang method call.)

#### SPI

The SPI (Service Provider Interface) for the plugin is the method that returns manifest and the manifest info itself. The method that returns manifest can be generated by `pluginpkg`, so implementing SPI work for the developer is to choose and construct different manifest info (`pluginpkg` also helps with this).

`Manifest` is the base struct for all other sub manifests. The caller can use `Kind` and `DeclareXXManifest` to convert them to sub manifests.

Manifest provides common metadata:

- Kind: the plugin's category. Now we have the audit kind, authentication kind, and so on. It's easy to add more.
- Name: name of the plugin, which is used to identify a plugin, so it cannot duplicate with other plugins.
- Version: we can load multiple versions a plugin into TiDB, but just activate one of them to support hot-fix or hot-upgrade.
- RequireVersion: it will make a simple relationship between different plugins.
- SysVars: it defines the plugin's configuration info.

Manifest also provides three lifecycle extension points: 

- Validate: called after loading all plugins but before onInit, so it can do cross plugins check before init.
- OnInit: plugin can prepare resource before real work using OnInit.
- OnShutDown: plugin can clean up its resources before dying using OnShutDown.

So we can image a common manifest code like this:

```
type Manifest struct {
    Kind           Kind
    Name           string
    Description    string
    Version        uint16
    RequireVersion map[string]uint16
    License        string
    BuildTime      string
    SysVars        map[string]*variable.SysVar
    Validate       func(ctx context.Context, manifest *Manifest) error
    OnInit         func(ctx context.Context) error
    OnShutdown     func(ctx context.Context) error
}
```

Base on `Kind`, we can define other subManifest for authentication plugin, audit plugin and so on.

Every subManifest will have a `Manifest` anonymous field as the FIRST field in struct definition, so every subManifest can be used as `Manifest` (by `unsafe.Pointer` cast). For example, an audit plugin' manifest will be like this:

```
type AuditManifest struct {
    Manifest
    NotifyEvent func(ctx context.Context) error
}
```

The reason we chose the embedded struct + unsafe.Pointer cast instead of the interface way here is that the first way is more flexible and more efficient to access data member than the fixed interface. At last, we also provide the package tools and a helper method to hide those details from the plugin developer.

#### Package tool

In this proposal, we add a simple tool `cmd/pluginpkg` to help package a plugin, and also uniform the package format.

Plugin's development event no longer needs to care about previous Manifest and so on, so the developer can just provide a `manifest.toml` configuration file like this in the package:

```
name = "conn_ip_example"
kind = "Audit"
description = "just a test"
version = "2"
license = ""
sysVars = [
    {name="conn_ip_example_test_variable", scope="Global", value="2"},
    {name="conn_ip_example_test_variable2", scope="Session", value="2"},
]
validate = "Validate"
onInit = "OnInit"
onShutdown = "OnShutdown"
export = [
    {extPoint="NotifyEvent", impl="NotifyEvent"}
]
```

- `name`: name of the plugin. It must be unique in the loaded TiDB instance.
- `kind`: kind of plugin. It determines the call-point in TiDB. The package tool is also based on it to generate a different manifest.
- `version`: the version of a plugin. For the same plugin, the same version is only loaded once.
- `description`: description of plugin usage.
- `license`: license of the plugin, which will display in `show plugins`.
- `sysVars`: it defines the variable needed by this plugin with name, scope and default value.
- `validate`: it specifies the callback function used to validate before load, e.g. auth plugin check `-with-skip-grant-tables` configuration.
- `onInit`: it specifies the callback function used to init plugin before it joins real work.
- `onShutdown`: the callback function will be called when the plugin shuts down to release outer resource held by the plugin, normally TiDB shutdown.
- `export`: it defines the callback list for the special kind plugins, e.g. for auth plugin it uses a `NotifyEvent` method to implement the `notifyEvent` extension point.

`pluginpkg` generates code and the generated code is built as a Go plugin, and using plugin package we also control the plugin binary's format:

- The plugin file name is `[pluginName]-[version].so`, so we can know the plugin's version from the filename.
- `pluginpath` will be `[pluginName]-[version]`, and then we can load the same plugin of a different version in the same host program.
- The package tool also adds some build time and misc info into Manifest info.

Package tools add an abstract layer over manifest, so we can change manifest easier in future if needed. 

#### Plugin Point

In TiDB code, we can add a new plugin point everywhere and:

- Call `plugin.GetByKind` or `plugin.Get` to find matched plugins.
- Call `plugin.Declare[Kind]Manifest` to cast Manifest to a special kind.
- Call the extension point method for special manifest.

We can see a simple example in `clientConn#Run` and `conn_ip_example` plugin implementation. 

Adding the new plugin point needs to modify the TiDB code and pass the required context and parameters.

#### Configuration

Every plugin has its own configurations. TiDB plugin uses system variables to handle configuration management requirement.

In `manifest.toml`, we can use the `sysVar` field to provide plugin's variable name and its default value. Plugin's system variable will be registered as TiDB system variable, so the user can read/modify variable just like normal system variables.

Plugin's variable name must use plugin name as the prefix. At last, the plugin cannot be reloaded if we change the plugin's sysVar (include default-value, add or remove variable).

We implement it by adding the plugin variable into `variable.SysVars` before `bootstrap`, so later `doDMLWorker` will handle them just as normal sysVars, and change `loadCommonGlobalVarsSQL` to load them. (that it cannot unload plugin and cannot modify sysVar during reload makes this implementation easier) 

#### Dependency

Go's plugin mechanism will check all dependency package hash to ensure link time and run time use the same version([see code](https://github.com/golang/go/blob/50bd1c4d4eb4fac8ddeb5f063c099daccfb71b26/src/runtime/plugin.go#L52)), so we no longer need to care about compiling package dependency.

But for the real world, there may be logic dependency between plugins. For example, some guy writes an authorization plugin but it relies on vault plugin and only works when vault is enabled but does not directly rely on the vault plugin's source code.

In `manifest.toml`, we can use `requireVersion` to declare A plugin requires B plugin in X version, and then plugin runtime will check it during the load or reload phase.

### Reload

Go plugin doesn't support unloading a plugin, but this cannot stop us from loading multiple versions of the plugin into the host program and framework, to ensure the last reloaded one will be active, and others aren't unloaded but disabled.

So, we can reload the plugin with a different version that is packaged by `pluginpkg` to modify the plugin's implementation logic. Although we can not change the plugin's meta info (e.g. sysVars) now, it's still useful.

#### Management

To add a plugin to TiDB, we need to:

- Add `-plugin-dir` as the start argument to specify the folder containing plugins, e.g. '-plugin-dir=/data/deploy/tidb/plugin'.
- Add `-plugin-load` as the start argument to specify the plugin id (name "-" version) that needs to be loaded, e.g. '-plugin-load=conn_limit-1'.

Then starting TiDB will load and enable plugins.

We can see all the plugins info by:

```
mysql> show plugins;
+-----------------+--------+-------+----------------------------------------------------+---------+---------+
| Name            | Status | Type  | Library                                            | License | Version |
+-----------------+--------+-------+----------------------------------------------------+---------+---------+
| conn_limit-1    | Ready  | Audit | /data/deploy/tidb/plugin/conn_limit-1.so           |         | 1       |
+-----------------+--------+-------+----------------------------------------------------+---------+---------+
1 row in set (0.00 sec)
```

To reload a loaded plugin, just use

```
mysql> admin plugins reload conn_limit-2;
```

### Limitations

The TiDB plugin has the following limitations:

- The plugin cannot be unloaded. Once the plugin is loaded into TiDB, it can never be unloaded until the server is restarted, but we can reload the plugin in the limited situation to the hotfix plugin bug.
- Read sysVars in OnInit will get unexpected value but can access `Manifest` to get the default config value.
- Reloading cannot change the sysVar's default value or add/remove the variable. 
- Building the plugin needs the TiDB source code tree, which is different from MySQL that can build plugin standalone (expect Information Schema and Storage Engine plugins)
- The plugin can only be written in Go.