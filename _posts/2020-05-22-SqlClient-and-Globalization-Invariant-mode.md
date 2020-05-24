---
layout: post
title:  SqlClient and Globalization Invariant mode
date:   2020-05-23 22:00:00 +0100
---
## The Runtime

In .NET Core 2.0 a runtime configuration option called [Globalization Invariant](https://github.com/dotnet/runtime/blob/master/docs/design/features/globalization-invariant-mode.md) mode was [announced](https://github.com/dotnet/announcements/issues/20). The purpose of this mode is to allow the .net core runtime to avoid using globalization apis which removes the requirement of packaging several parts of the managed and native runtime dependencies and makes the deployment size lower. 

You can [set Globalization Invariant](https://github.com/dotnet/runtime/blob/master/docs/design/features/globalization-invariant-mode.md#app-behavior-with-and-without-the-invariant-config-switch) mode using a project property, runtimeconfig.json or setting an environment variable. You can [detect if it is enabled](https://github.com/dotnet/SqlClient/issues/81#issuecomment-487674672) at runtime by looking at a culture name to see if it contains the word invariant

```csharp
CultureInfo.GetCultureInfo("en-US").EnglishName.Contains("Invariant")
```

## The Library

Globalization Invariant mode causes a problem for the `Microsoft.Data.SqlClient` libraries because the library needs to read strings from the protocol and fetched data and convert them from the source encoding into the .NET strings in their correct encoding. Even if you don't intend to read any strings the act of connecting to the database requires that some strings like the default collation sequence identifier for the database are read. If you got past the collation sequence by hardcoding a default as soon as you needed to decode column names you'd hit the same problem. The TDS protocol used in SqlServer requires certain strings and those can't be safely decoded so query functionality cannot work.

The `System.Data.SqlClient` version of the library that ships with .NET Core if you try to connect to an sql database with the runtime in invariant mode you will receive a cryptic `InvalidOperationException` with the message `Internal connection fatal error`.

The `Microsoft.Data.SqlClient` version of the library was updated to check for invariant mode when you try top open a connection and throw a `NotSupportedException` with the message `Globalization Invariant Mode is not supported.`

This is done when you try to open the connection so that you can safely reference the library and construct types if you needed to, for example use by an ORM, but that as soon as you start to do something which is clearly going to fail it is signalled to you. If you hit this message your app should quit because there is no way to change the invariant mode of a process once it has started running.

## The Environment

Globalization Invariant mode can be enabled on any OS but you will probably find it on docker linux containers and IOT builds where storage and runtime memory size are constrained. This problem is mostly found on Alpine linux containers where the native runtime size is made as small as possible by removing ICU libraries and putting the runtime into invariant mode. This results in small and fast to build images but will leave you unable to connect to SQL Server.

## The Resolution

To change this you can:
* opt to use a larger container which contains the required ICU libraries and remove the invariant mode setting if you applied one. 
* install the ICU libraries as part of your dockerfile
```
    RUN apk add --no-cache icu-libs
    ENV DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=false
```

