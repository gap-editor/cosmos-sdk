---
sidebar_position: 1
---

# AutoCLI

:::note Synopsis
This document details how to build CLI and REST interfaces for a module. Examples from various Cosmos SDK modules are included.
:::

:::note Pre-requisite Readings

* [CLI](https://docs.cosmos.network/main/core/cli)

:::

The `autocli` (also known as `client/v2`) package is a [Go library](https://pkg.go.dev/cosmossdk.io/client/v2/autocli) for generating CLI (command line interface) interfaces for Cosmos SDK-based applications. It provides a simple way to add CLI commands to your application by generating them automatically based on your gRPC service definitions. Autocli generates CLI commands and flags directly from your protobuf messages, including options, input parameters, and output parameters. This means that you can easily add a CLI interface to your application without having to manually create and manage commands.

## Overview

`autocli` generates CLI commands and flags for each method defined in your gRPC service. By default, it generates commands for each gRPC services. The commands are named based on the name of the service method.

For example, given the following protobuf definition for a service:

```protobuf
service MyService {
  rpc MyMethod(MyRequest) returns (MyResponse) {}
}
```

For instance, `autocli` would generate a command named `my-method` for the `MyMethod` method. The command will have flags for each field in the `MyRequest` message.

It is possible to customize the generation of transactions and queries by defining options for each service.

## Application Wiring

Here are the steps to use AutoCLI:

1. Ensure your app's modules implements the `appmodule.AppModule` interface.
2. (optional) Configure how behave `autocli` command generation, by implementing the `func (am AppModule) AutoCLIOptions() *autocliv1.ModuleOptions` method on the module.
3. Use the `autocli.AppOptions` struct to specify the modules you defined. If you are using `depinject`, it can automatically create an instance of `autocli.AppOptions` based on your app's configuration.
4. Use the `EnhanceRootCommand()` method provided by `autocli` to add the CLI commands for the specified modules to your root command.

:::tip
AutoCLI is additive only, meaning _enhancing_ the root command will only add subcommands that are not already registered. This means that you can use AutoCLI alongside other custom commands within your app.
:::

Here's an example of how to use `autocli` in your app:

``` go
// Define your app's modules
testModules := map[string]appmodule.AppModule{
    "testModule": &TestModule{},
}

// Define the autocli AppOptions
autoCliOpts := autocli.AppOptions{
    Modules: testModules,
}

// Create the root command
rootCmd := &cobra.Command{
    Use: "app",
}

if err := appOptions.EnhanceRootCommand(rootCmd); err != nil {
    return err
}

// Run the root command
if err := rootCmd.Execute(); err != nil {
    return err
}
```

### Keyring

`autocli` uses a keyring for key name resolving names and signing transactions.

:::tip
AutoCLI provides a better UX than normal CLI as it allows to resolve key names directly from the keyring in all transactions and commands.

```sh
<appd> q bank balances alice
<appd> tx bank send alice bob 1000denom
```

:::

The keyring used for resolving names and signing transactions is provided via the `client.Context`.
The keyring is then converted to the `client/v2/autocli/keyring` interface.
If no keyring is provided, the `autocli` generated command will not be able to sign transactions, but will still be able to query the chain.

:::tip
The Cosmos SDK keyring and Hubl keyring both implement the `client/v2/autocli/keyring` interface, thanks to the following wrapper:

```go
keyring.NewAutoCLIKeyring(kb)
```

:::

## Signing

`autocli` supports signing transactions with the keyring.
The [`cosmos.msg.v1.signer` protobuf annotation](https://docs.cosmos.network/main/build/building-modules/protobuf-annotations) defines the signer field of the message.
This field is automatically filled when using the `--from` flag or defining the signer as a positional argument.

:::warning
AutoCLI currently supports only one signer per transaction.
:::

## Module wiring & Customization

The `AutoCLIOptions()` method on your module allows to specify custom commands, sub-commands or flags for each service, as it was a `cobra.Command` instance, within the `RpcCommandOptions` struct. Defining such options will customize the behavior of the `autocli` command generation, which by default generates a command for each method in your gRPC service.

```go
*autocliv1.RpcCommandOptions{
  RpcMethod: "Params", // The name of the gRPC service
  Use:       "params", // Command usage that is displayed in the help
  Short:     "Query the parameters of the governance process", // Short description of the command
  Long:      "Query the parameters of the governance process. Specify specific param types (voting|tallying|deposit) to filter results.", // Long description of the command
  PositionalArgs: []*autocliv1.PositionalArgDescriptor{
    {ProtoField: "params_type", Optional: true}, // Transform a flag into a positional argument
  },
}
```

:::tip
AutoCLI can create a gov proposal of any tx by simply setting the `GovProposal` field to `true` in the `autocli.RpcCommandOptions` struct.
Users can however use the `--no-proposal` flag to disable the proposal creation (which is useful if the authority isn't the gov module on a chain).
:::

### Specifying Subcommands

By default, `autocli` generates a command for each method in your gRPC service. However, you can specify subcommands to group related commands together. To specify subcommands, use the `autocliv1.ServiceCommandDescriptor` struct.

This example shows how to use the `autocliv1.ServiceCommandDescriptor` struct to group related commands together and specify subcommands in your gRPC service by defining an instance of `autocliv1.ModuleOptions` in your `autocli.go`.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-beta.0/x/gov/autocli.go#L94-L97
```

### Positional Arguments

By default `autocli` generates a flag for each field in your protobuf message. However, you can choose to use positional arguments instead of flags for certain fields.

To add positional arguments to a command, use the `autocliv1.PositionalArgDescriptor` struct, as seen in the example below. Specify the `ProtoField` parameter, which is the name of the protobuf field that should be used as the positional argument. In addition, if the parameter is a variable-length argument, you can specify the `Varargs` parameter as `true`. This can only be applied to the last positional parameter, and the `ProtoField` must be a repeated field.

Here's an example of how to define a positional argument for the `Account` method of the `auth` service:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-beta.0/x/auth/autocli.go#L25-L30
```

Then the command can be used as follows, instead of having to specify the `--address` flag:

```bash
<appd> query auth account cosmos1abcd...xyz
```

#### Flattened Fields in Positional Arguments

AutoCLI also supports flattening nested message fields as positional arguments. This means you can access nested fields
using dot notation in the `ProtoField` parameter. This is particularly useful when you want to directly set nested
message fields as positional arguments.

For example, if you have a nested message structure like this:

```protobuf
message Permissions {
    string level = 1;
    repeated string limit_type_urls = 2;
}

message MsgAuthorizeCircuitBreaker {
    string grantee = 1;
    Permissions permissions = 2;
}
```

You can flatten the fields in your AutoCLI configuration:

```go
{
    RpcMethod: "AuthorizeCircuitBreaker",
    Use:       "authorize <grantee> <level> <msg_type_urls>",
    PositionalArgs: []*autocliv1.PositionalArgDescriptor{
        {ProtoField: "grantee"},
        {ProtoField: "permissions.level"},
        {ProtoField: "permissions.limit_type_urls"},
    },
}
```

This allows users to provide values for nested fields directly as positional arguments:

```bash
<appd> tx circuit authorize cosmos1... super-admin "/cosmos.bank.v1beta1.MsgSend,/cosmos.bank.v1beta1.MsgMultiSend"
```

Instead of having to provide a complex JSON structure for nested fields, flattening makes the CLI more user-friendly by allowing direct access to nested fields.

#### Customising Flag Names

By default, `autocli` generates flag names based on the names of the fields in your protobuf message. However, you can customise the flag names by providing a `FlagOptions`. This parameter allows you to specify custom names for flags based on the names of the message fields.

For example, if you have a message with the fields `test` and `test1`, you can use the following naming options to customise the flags:

``` go
autocliv1.RpcCommandOptions{ 
    FlagOptions: map[string]*autocliv1.FlagOptions{ 
        "test": { Name: "custom_name", }, 
        "test1": { Name: "other_name", }, 
    }, 
}
```

`FlagsOptions` is defined like sub commands in the `AutoCLIOptions()` method on your module.

### Combining AutoCLI with Other Commands Within A Module

AutoCLI can be used alongside other commands within a module. For example, the `gov` module uses AutoCLI to generate commands for the `query` subcommand, but also defines custom commands for the `proposer` subcommands.

In order to enable this behavior, set in `AutoCLIOptions()` the `EnhanceCustomCommand` field to `true`, for the command type (queries and/or transactions) you want to enhance.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/fa4d87ef7e6d87aaccc94c337ffd2fe90fcb7a9d/x/gov/autocli.go#L98
```

If not set to true, `AutoCLI` will not generate commands for the module if there are already commands registered for the module (when `GetTxCmd()` or `GetTxCmd()` are defined).

### Skip a command

AutoCLI automatically skips unsupported commands when [`cosmos_proto.method_added_in` protobuf annotation](https://docs.cosmos.network/main/build/building-modules/protobuf-annotations) is present.

Additionally, a command can be manually skipped using the `autocliv1.RpcCommandOptions`:

```go
*autocliv1.RpcCommandOptions{
  RpcMethod: "Params", // The name of the gRPC service
  Skip: true,
}
```

### Use AutoCLI for non module commands

It is possible to use `AutoCLI` for non module commands. The trick is still to implement the `appmodule.Module` interface and append it to the `appOptions.ModuleOptions` map.

For example, here is how the SDK does it for `cometbft` gRPC commands:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/client/v2.0.0-beta.1/client/grpc/cmtservice/autocli.go#L52-L71
```

## Summary

`autocli` let you generate CLI to your Cosmos SDK-based applications without any cobra boilerplate. It allows you to easily generate CLI commands and flags from your protobuf messages, and provides many options for customising the behavior of your CLI application.

To further enhance your CLI experience with Cosmos SDK-based blockchains, you can use `hubl`. `hubl` is a tool that allows you to query any Cosmos SDK-based blockchain using the new AutoCLI feature of the Cosmos SDK. With `hubl`, you can easily configure a new chain and query modules with just a few simple commands.

For more information on `hubl`, including how to configure a new chain and query a module, see the [Hubl documentation](https://docs.cosmos.network/main/tooling/hubl).
