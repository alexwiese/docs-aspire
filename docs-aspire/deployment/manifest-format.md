---
title: .NET Aspire manifest format for deployment tool builders
description: Learn about the .NET Aspire manifest format in this comprehensive deployment tool builder guide.
ms.date: 11/10/2023
ms.topic: reference
---

# .NET Aspire manifest format for deployment tool builders

In this article, you learn about the .NET Aspire manifest format. This article serves as a reference guide for deployment tool builders, aiding in the creation of tooling to deploy .NET Aspire applications on specific hosting platforms, whether on-premises or in the cloud.

.NET Aspire simplifies the local development experience by helping to manage interdependencies between application components. To help simplify the deployment of applications, .NET Aspire projects can generate a "manifest" of all the resources defined as a JSON formatted file.

## Generate a manifest

A valid .NET Aspire application is required to generate a manifest. To get started, create
a .NET Aspire application using the `aspire-starter` .NET template:

```dotnetcli
dotnet new aspire-starter --use-redis-cache `
    -n AspireApp && `
    cd AspireApp
```

Manifest generation is achieved by running `dotnet build` with a special target:

```dotnetcli
dotnet run --project AspireApp.AppHost\AspireApp.AppHost.csproj `
    -- `
    --publisher manifest `
    --output-path aspire-manifest.json
```

For more information, see [dotnet build](../../core/tools/dotnet-build.md). The previous command produces the following output:

```Output
Building...
info: Aspire.Hosting.Publishing.ManifestPublisher[0]
      Published manifest to: .\AspireApp.AppHost\aspire-manifest.json
```

The file generated is the .NET Aspire manifest and is used by tools to support deploying into target cloud environments.

## Basic manifest format

Publishing the manifest from the default starter template for .NET Aspire produces the following JSON output:

```json
{
  "resources": {
    "cache": {
      "type": "redis.v0"
    },
    "apiservice": {
      "type": "project.v0",
      "path": "AspireApp.ApiService\\AspireApp.ApiService.csproj",
      "env": {
        "OTEL_DOTNET_EXPERIMENTAL_OTLP_EMIT_EXCEPTION_LOG_ATTRIBUTES": "true",
        "OTEL_DOTNET_EXPERIMENTAL_OTLP_EMIT_EVENT_LOG_ATTRIBUTES": "true"
      },
      "bindings": {
        "http": {
          "scheme": "http",
          "protocol": "tcp",
          "transport": "http"
        },
        "https": {
          "scheme": "https",
          "protocol": "tcp",
          "transport": "http"
        }
      }
    },
    "webfrontend": {
      "type": "project.v0",
      "path": "AspireApp.Web\\AspireApp.Web.csproj",
      "env": {
        "OTEL_DOTNET_EXPERIMENTAL_OTLP_EMIT_EXCEPTION_LOG_ATTRIBUTES": "true",
        "OTEL_DOTNET_EXPERIMENTAL_OTLP_EMIT_EVENT_LOG_ATTRIBUTES": "true",
        "ConnectionStrings__cache": "{cache.connectionString}",
        "services__apiservice__0": "{apiservice.bindings.http.url}",
        "services__apiservice__1": "{apiservice.bindings.https.url}"
      },
      "bindings": {
        "http": {
          "scheme": "http",
          "protocol": "tcp",
          "transport": "http"
        },
        "https": {
          "scheme": "https",
          "protocol": "tcp",
          "transport": "http"
        }
      }
    }
  }
}
```

The manifest format JSON consists of a single object called `resources`, which contains a property for each resource specified in _Program.cs_ (the `name` argument for each name is used as the property for each of the child resource objects in JSON).

### Connection string and binding references

In the previous example, there are two project resources and one Redis cache resource. The _webfrontend_ depends on both the _apiservice_ (project) and _cache_ (Redis) resources.

This dependency is known because the environment variables for the _webfrontend_ contain placeholders that reference the two other resources:

```json
"env": {
  // ... other environment variables omitted for clarity
  "ConnectionStrings__cache": "{cache.connectionString}",
  "services__apiservice__0": "{apiservice.bindings.http.url}",
  "services__apiservice__1": "{apiservice.bindings.https.url}"
},
```

The `apiservice` resource is referenced by `webfrontend` using the call `WithReference(apiservice)` in the app host _Program.cs_ file and `redis` is referenced using the call `WithReference(cache)`:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

var cache = builder.AddRedisContainer("cache");

var apiservice = builder.AddProject<Projects.AspireApp_ApiService>("apiservice");

builder.AddProject<Projects.AspireApp_Web>("webfrontend")
    .WithReference(cache)
    .WithReference(apiservice);

builder.Build().Run();
```

References between project resource types result in service discovery variables being injected into the referencing project. References to
well know reference types such as Redis result in connection strings being injected.

:::image type="content" source="media/manifest-placeholder-strings.png" lightbox="media/manifest-placeholder-strings.png" alt-text="A diagram showing which resources contribute to which corresponding placeholder strings.":::

For more information on how resources in the app model and references between them work, see, [.NET Aspire orchestration overview](../app-host-overview.md).

### Placeholder string structure

Placeholder strings reference the structure of the .NET Aspire manifest:

:::image type="content" source="media/placeholder-mappings.png" lightbox="media/placeholder-mappings.png" alt-text="A diagram showing how the manifest JSON structure maps to placeholder strings.":::

The final segment of the placeholder string (`url` in this case) is generated by the tool processing the manifest. There are several suffixes that could be used on the placeholder string:

- `connectionString`: For well-known resource types such as Redis. Deployment tools translate the resource in the most appropriate infrastructure for the target cloud environment and then produce a .NET Aspire compatible connection string for the consuming application to use.
- `url`: For service-to-service references where a well-formed URL is required. The deployment tool produces the `url` based on the scheme, protocol, and transport defined in the manifest and the underlying compute/networking topology that was deployed.
- `host`: The host segment of the URL.
- `port`: The port segment of the URL.

## Resource types

Each resource has a `type` field. When a deployment tool reads the manifest, it should read
the type to verify whether it can correctly process the manifest. During the .NET Aspire preview period, all resource types have a `v0` suffix to indicate that they're subject to change. As
.NET Aspire approaches release a `v1` suffix will be used to signify that the structure of the
manifest for that resource type should be considered stable (subsequent updates increment
the version number accordingly).

### Common resource fields

The `type` field is the only field that is common across all resource types, however, the
`project.v0`, `container.v0`, and `executable.v0` resource types also share the `env` and `bindings`
fields.

> [!NOTE]
> The `executable.v0` resource type isn't fully implemented in the manifest due to its lack of
utility in deployment scenarios.

The `env` field type is a basic key/value mapping where the values might contain [_placeholder strings_](#placeholder-string-structure).

Bindings are specified in the `bindings` field with each binding contained within its own field under the `bindings` JSON object. The fields committed by the .NET aspire include:

- `scheme`: One of the following values `tcp`, `udp`, `http`, or `https`.
- `protocol`: One of the following values `tcp` or `udp`
- `transport`: Same as `scheme`, but used to disambiguate between `http` and `http2`.
- `containerPort`: Optional, if omitted defaults to port 80.

## Built-in resources

The following table is a list of resource types that are explicitly generated by .NET Aspire and
extensions developed by the .NET Aspire team:

### Cloud agnostic resource types

These resources are available in the _Aspire.Hosting_ package.

| App model usage | Manifest resource type | Heading link |
|--|--|--|
| `AddProject<T>(...)` | `project.v0` | [Project resource type](#project-resource-type) |
| `AddContainer(...)` | `container.v0` | [Container resource type](#container-resource-type) |
| `AddPostgresConnection(...)` | `postgres.connection.v0` | [Postgres resource types](#postgres-resource-types) |
| `AddPostgresContainer(...)` | `postgres.server.v0` | [Postgres resource types](#postgres-resource-types) |
| `AddPostgresContainer(...).AddDatabase(...)` | `postgres.database.v0` | [Postgres resource types](#postgres-resource-types) |
| `AddRabbitMQConnection(...)` | `rabbitmq.connection.v0` | [RabbitMQ resource types](#rabbitmq-resource-types) |
| `AddRabbitMQContainer(...)` | `rabbitmq.server.v0` | [RabbitMQ resource types](#rabbitmq-resource-types) |
| `AddRedisConnection(...)`<br>`AddRedis(...)` | `redis.v0` | [Redis resource type](#redis-resource-type) |
| `AddSqlServerConnection(...)` | `sqlserver.connection.v0` | [SQL Server resource types](#sql-server-resource-types) |
| `AddSqlServerContainer(...)` | `sqlserver.server.v0` | [SQL Server resource types](#sql-server-resource-types) |
| `AddSqlServerContainer(...).AddDatabase(...)` | `sqlserver.database.v0` | [SQL Server resource types](#sql-server-resource-types) |

#### Project resource type

Example code:

```csharp
var builder = DistributedApplication.CreateBuilder(args);
var apiservice = builder.AddProject<Projects.AspireApp_ApiService>("apiservice");
```

Example manifest:

```json
"apiservice": {
    "type": "project.v0",
    "path": "..\\AspireApp.ApiService\\AspireApp.ApiService.csproj",
    "env": {
        "OTEL_DOTNET_EXPERIMENTAL_OTLP_EMIT_EXCEPTION_LOG_ATTRIBUTES": "true",
        "OTEL_DOTNET_EXPERIMENTAL_OTLP_EMIT_EVENT_LOG_ATTRIBUTES": "true"
    },
    "bindings": {
        "http": {
            "scheme": "http",
            "protocol": "tcp",
            "transport": "http"
        },
        "https": {
            "scheme": "https",
            "protocol": "tcp",
            "transport": "http"
        }
    }
}
```

#### Container resource type

Example code:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

builder.AddContainer("mycontainer", "myimage")
       .WithEnvironment("LOG_LEVEL", "WARN")
       .WithServiceBinding(3000, scheme: "http");
```

Example manifest:

```json
{
  "resources": {
    "mycontainer": {
      "type": "container.v0",
      "image": "myimage:latest",
      "env": {
        "LOG_LEVEL": "WARN"
      },
      "bindings": {
        "http": {
          "scheme": "http",
          "protocol": "tcp",
          "transport": "http",
          "containerPort": 3000
        }
      }
    }
  }
}
```

#### Postgres resource types

Example code:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

builder.AddPostgresConnection(
    "postgres1",
    "Host=mypgserver;Port=5432;Database=inventory;Username=myUsername;Password=myPassword");

builder.AddPostgresContainer("postgres2").AddDatabase("shipping");
```

Example manifest:

```json
{
  "resources": {
    "postgres1": {
      "type": "postgres.connection.v0",
      "connectionString": "Host=mypgserver;Port=5432;Database=inventory;Username=myUsername;Password=myPassword"
    },
    "postgres2": {
      "type": "postgres.server.v0"
    },
    "shipping": {
      "type": "postgres.database.v0",
      "parent": "postgres2"
    }
  }
}
```

#### RabbitMQ resource types

RabbitMQ has two resource types, `rabbitmq.server.v0` and `rabbitmq.connectionv0`. The following
sample shows how they're added to the app model.

```csharp
var builder = DistributedApplication.CreateBuilder(args);

builder.AddRabbitMQContainer("rabbitmq1");
builder.AddRabbitMQConnection("rabbitmq2", "amqp://guest:guest@localhost:5672");
```

The previous code produces the following manifest:

```json
{
  "resources": {
    "rabbitmq1": {
      "type": "rabbitmq.server.v0"
    },
    "rabbitmq2": {
      "type": "rabbitmq.connection.v0",
      "connectionString": "amqp://guest:guest@localhost:5672"
    }
  }
}
```

The second variation is used when there's already an existing Rabbit MQ server. The first
variation is used when the deployment tool should deploy a new service that supports the
Rabbit MQ protocol.

#### Redis resource type

Example code:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

builder.AddRedis("redis1", "myredis:6379");
builder.AddRedisContainer("redis2");
```

Example manifest:

```json
{
  "resources": {
    "redis1": {
      "type": "redis.v0",
      "connectionString": "myredis:6379"
    },
    "redis2": {
      "type": "redis.v0"
    }
  }
}
```

#### SQL Server resource types

Example code:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

builder.AddSqlServerConnection(
    "sql1",
    "Server=myexistingserver;Database=inventory;Trusted_Connection=True;");

builder.AddSqlServerContainer("sql2").AddDatabase("shipping");
```

Example manifest:

```json
{
  "resources": {
    "sql1": {
      "type": "sqlserver.connection.v1",
      "connectionString": "Server=myexistingserver;Database=inventory;Trusted_Connection=True;"
    },
    "sql2": {
      "type": "sqlserver.server.v1"
    },
    "shipping": {
      "type": "sqlserver.database.v1",
      "parent": "sql2"
    }
  }
}
```

### Azure-specific resource types

The following resources are available in the [Aspire.Hosting.Azure](https://www.nuget.org/packages/Aspire.Hosting.Azure) NuGet package.

| App Model usage | Manifest resource type | Heading link |
|--|--|--|
| `AddAzureKeyVault(...)` | `azure.keyvault.v0` | [Azure Key Vault resource type](#azure-key-vault-resource-type) |
| `AddAzureServiceBus(...)` | `azure.servicebus.v0` | [Azure Service Bus resource type](#azure-service-bus-resource-type) |
| `AddAzureStorage(...)` | `azure.storage.v0` | [Azure Storage resource types](#azure-storage-resource-types) |
| `AddAzureStorage(...).AddBlobs(...)` | `azure.storage.blob.v0` | [Azure Storage resource types](#azure-storage-resource-types) |
| `AddAzureStorage(...).AddTables(...)` | `azure.storage.table.v0` | [Azure Storage resource types](#azure-storage-resource-types) |
| `AddAzureStorage(...).AddQueues(...)` | `azure.storage.queue.v0` | [Azure Storage resource types](#azure-storage-resource-types) |
| `AddAzureRedis(...)` | `azure.redis.v0` | [Azure Redis resource types](#azure-redis-resource-type) |
| `AddAzureAppConfiguration(...)` | `azure.appconfiguration.v0` | [Azure App Configuration resource types](#azure-app-configuration-resource-type) |
| `AddAzureSqlServer(...)` | `azure.sql.v0` | [Azure SQL resource types](#azure-sql-resource-types) |
| `AddAzureSqlServer(...).AddDatabase(...)` | `azure.sql.database.v0` | [Azure SQL resource types](#azure-sql-resource-types) |

#### Azure Key Vault resource type

Example code:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

builder.AddAzureKeyVault("keyvault1");
```

Example manifest:

```json
{
  "resources": {
    "keyvault1": {
      "type": "azure.keyvault.v0"
    }
  }
}
```

#### Azure Service Bus resource type

Example code:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

builder.AddAzureServiceBus(
    "sb1",
    queueNames: new[] { "queue1", "queue2" },
    topicNames: new[] { "topic1", "topic2" }
);
```

Example manifest:

```json
{
  "resources": {
    "sb1": {
      "type": "azure.servicebus.v0",
      "queues": [
        "queue1",
        "queue2"
      ],
      "topics": [
        "topic1",
        "topic2"
      ]
    }
  }
}
```

#### Azure Storage resource types

Example code:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

var storage = builder.AddAzureStorage("images");

storage.AddBlobs("blobs");
storage.AddQueues("queues");
storage.AddTables("tables");
```

Example manifest:

```json
{
  "resources": {
    "images": {
      "type": "azure.storage.v0"
    },
    "blobs": {
      "type": "azure.storage.blob.v0",
      "parent": "images"
    },
    "queues": {
      "type": "azure.storage.queue.v0",
      "parent": "images"
    },
    "tables": {
      "type": "azure.storage.table.v0",
      "parent": "images"
    }
  }
}
```

#### Azure Redis resource type

Example code:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

builder.AddAzureRedis("azredis1");
```

Example manifest:

```json
{
  "resources": {
    "azredis1": {
      "type": "azure.redis.v0"
    }
 }
}
```

#### Azure App Configuration resource type

Example code:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

builder.AddAzureAppConfiguration("appconfig1");
```

Example manifest:

```json
{
  "resources": {
    "appconfig1": {
      "type": "azure.appconfiguration.v0"
    }
  }
}
```

#### Azure SQL resource types

Example code:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

builder.AddAzureSqlServer("sql1").AddDatabase("inventory");
```

Example manifest:

```json
{
  "resources": {
    "sql1": {
      "type": "azure.sql.v0"
    },
    "inventory": {
      "type": "azure.sql.database.v0",
      "parent": "sql1"
    }
  }
}
```

## See also

- [.NET Aspire overview](../get-started/aspire-overview.md)
- [.NET Aspire orchestration overview](../app-host-overview.md)
- [.NET Aspire components overview](../components-overview.md)
- [Service discovery in .NET Aspire](../service-discovery/overview.md)