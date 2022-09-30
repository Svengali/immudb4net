# ImmuDB4Net - The Official [immudb] Client for .NET [![License](https://img.shields.io/github/license/codenotary/immudb4j)](LICENSE)

[immudb]: https://immudb.io/

## Contents

- [1. Introduction](#1-introduction)
- [2. Prerequisites](#2-prerequisites)
- [3. Installation](#3-installation)
- [4. Build locally from sources](#4-build-locally-from-sources)
- [5. Supported Versions](#5-supported-versions)
- [6. Quickstart](#6-quickstart)
- [7. Step-by-step Guide](#7-step-by-step-guide)
  - [7.1. Creating a Client](#71-creating-a-client)
  - [7.2. User Sessions](#72-user-sessions)
  - [7.3. Creating a Database](#73-creating-a-database)
  - [7.4. Setting the Active Database](#74-setting-the-active-database)
  - [7.5. Standard Read and Write](#75-standard-read-and-write)
  - [7.6. Verified or Safe Read and Write](#76-verified-or-safe-read-and-write)
  - [7.7. Multi-key Read and Write](#77-multi-key-read-and-write)
  - [7.8. Executing SQL statements and queries](#78-executing-sql-statements-and-queries)
  - [7.9. Closing the client](#79-closing-the-client)
- [8. Building from source](#8-building-from-source)
- [9. Contributing](#9-contributing)

## 1. Introduction

ImmuDB4Net implements a [gRPC] immudb client, based on [immudb's official protobuf definition].
It exposes a minimal and simple to use API for applications, while the cryptographic verifications and state update protocol implementation
are fully implemented internally by this client.

The latest validated immudb state may be kept in the local file system using default `FileImmuStateHolder`.
Please read [immudb Research Paper] for details of how immutability is ensured by [immudb].

[gRPC]: https://grpc.io/
[immudb Research Paper]: https://immudb.io/
[immudb's official protobuf definition](https://github.com/codenotary/immudb/blob/master/pkg/api/schema/schema.proto)

## 2. Prerequisites

ImmuDB4Net assumes you have access to a running immudb server.
Running `immudb` on your system is very simple, please refer to this [immudb QuickStart](https://docs.immudb.io/master/quickstart.html) page.

## 3. Installation

Include ImmuDB4Net as a dependency in your project via Nuget package. `ImmuDB4Net` is currently hosted on [NuGet.Org]

[NuGet.Org]: https://nuget.org

## 4. Build locally from sources

Use ```dotnet build``` to build locally the ImmuDB client assembly.

## 5. Supported Versions

ImmuDB4Net supports the [latest immudb server] release, that is 1.3.2 at the time of updating this document.

[latest immudb server]: https://github.com/codenotary/immudb/releases/tag/v1.3.2

## 6. Quickstart

[Hello Immutable World!] example can be found in `immudb-client-examples` repo:

[Hello Immutable World!]: https://github.com/codenotary/immudb-client-examples/blob/feat/dotnet-example/dotnet/simple-app

## 7. Step-by-step Guide

### 7.1. Creating a Client

The following code snippets show how to create a client.

Using default configuration:

``` C#
    ImmuClient immuClient = ImmuClient.NewBuilder().Build();

    // or

    Immuclient immuClient = new ImmuClient();
    Immuclient immuClient = new ImmuClient("localhost", 3322);
    Immuclient immuClient = new ImmuClient("localhost", 3322, "defaultdb");


```

Setting `immudb` url and port:

``` C#
    ImmuClient immuClient = ImmuClient.NewBuilder()
                                .WithServerUrl("localhost")
                                .WithServerPort(3322)
                                .Build();

    ImmuClient immuClient = ImmuClient.NewBuilder()
                                .WithServerUrl("localhost")
                                .WithServerPort(3322)
                                .Build();

```

Customizing the `State Holder`:

``` C#
    FileImmuStateHolder stateHolder = FileImmuStateHolder.NewBuilder()
                                        .WithStatesFolder("./my_immuapp_states")
                                        .Build();

    ImmuClient immuClient = ImmuClient.NewBuilder()
                                      .WithStateHolder(stateHolder)
                                      .Build();
```

### 7.2. User Sessions

Use `Open` and `Close` methods to initiate and terminate user sessions:

``` C#
    await immuClient.Open("usr1", "pwd1", "defaultdb");

    // Interact with immudb using logged-in user.
    //...

    await immuClient.Close();

    // or one liner open the session right 
    client = await ImmuClient.NewBuilder().Open();

    //then close it
    await immuClient.Close();

```

### 7.3. Creating a Database

Creating a new database is quite simple:

``` C#
    await immuClient.CreateDatabase("db1");
```

### 7.4. Setting the Active Database

Specify the active database with:

``` C#
    await immuClient.UseDatabase("db1");
```

### 7.5. Standard Read and Write

immudb provides standard read and write operations that behave as in a standard
key-value store i.e. no cryptographic verification is involved. Such operations
may be used when validations can be postponed.

``` C#
    await client.Set("k123", "v123");
    string v = await client.Get("k123").ToString();
or
    await client.Set("k123", new byte[]{1, 2, 3});
    byte[] v = await client.Get("k123").Value;
```

### 7.6. Verified or Safe Read and Write

immudb provides built-in cryptographic verification for any entry. The client
implements the mathematical validations while the application uses as a standard
read or write operation:

``` C#
    try 
    {
        await Client.VerifiedSet("k123", new byte[]{1, 2, 3});
        byte[] v = await client.VerifiedGet("k123").Value;
or
        await client.VerifiedSet("k123", "v123");
        string v = await client.VerifiedGet("k123").ToString();

    } catch(VerificationException e) {

        // Check if it is a data tampering detected case!

    }
```

### 7.7. Multi-key Read and Write

Transactional multi-key read and write operations are supported by immudb and ImmuDB4Net.

Atomic multi-key write (all entries are persisted or none):

``` C#
        List<KVPair> kvs = new List<KVPair>() 
        {
            new KVPair("sga-key1", new byte[] {1, 2}),
            new KVPair("sga-key2", new byte[] {3, 4})
        }

        try 
        {
            await immuClient.SetAll(kvs);
        } catch (CorruptedDataException e) 
        {
            // ...
        }
```

Atomic multi-key read (all entries are retrieved or none):

``` C#
    List<string> keys = new List<string>() {key1, key2, key3};
    List<Entry> result = await immuClient.GetAll(keys);

    foreach(Entry entry in result) {
        byte[] key = entry.Key;
        byte[] value = entry.Value;
        // ...
    }
```

### 7.8. Executing SQL statements and queries

ImmuDB4Net provides `SQLExec` and `SQLQuery` commands and an example of usage is below:

``` C#
        var client = new ImmuClient();
        await client.Open("immudb", "immudb", "defaultdb");

        await client.SQLExec("CREATE TABLE IF NOT EXISTS logs(id INTEGER AUTO_INCREMENT, created TIMESTAMP, entry VARCHAR, PRIMARY KEY id)");
        await client.SQLExec("CREATE INDEX IF NOT EXISTS ON logs(created)");
        var rspInsert = await client.SQLExec("INSERT INTO logs(created, entry) VALUES($1, $2)",
                SQLParameter.Create(DateTime.UtcNow),
                SQLParameter.Create("hello immutable world"));
        var queryResult = await client.SQLQuery("SELECT created, entry FROM LOGS order by created DESC");
        var sqlVal = queryResult.Rows[0]["entry"];
        
        Console.WriteLine($"The log entry is: {sqlVal.Value}");

```

### 7.9. Closing the client

Use `Close`, for closing the connection with immudb server . When terminating the process, use the `ImmuClient.ReleaseSdkResources` function :

``` C#
    await client.Close();
    await ImmuClient.ReleaseSdkResources();
```

Note: After the shutdown, a new client needs to be created to establish a new connection.

## 8. Building from source

To build from source you need as prerequisites to clone a local copy of the git repo: <https://github.com/codenotary/immudb4net>
and then to have installed on the build machine the dotnet 6.0 SDK. Then, from the terminal just run ```dotnet build``` .

In order to successfully execute the integration tests with the command```dotnet test``` you have to install as prerequisites ```docker``` and also to start locally an ImmuDB instance on port 3322. For example, you can run ImmuDB in docker as below:

``` bash
docker run -d --name immudb -p 3322:3322 codenotary/immudb:latest
```

## 9. Contributing

We welcome contributions. Feel free to join the team!

To report bugs or get help, use [GitHub's issues].

[GitHub's issues]: https://github.com/codenotary/immudb4net/issues
