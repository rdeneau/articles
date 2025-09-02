---
title: Applying object-oriented design principles - In-memory configuration builder
description: Applying object-oriented design principles - In-memory configuration builder
tags: #csharp #dotnet #oop
published: false
# published_at: 2025-09-01 15:43 +0200
---

Testing functionality that takes an XML or JSON file as input is no easy task. We'll see how to apply object-oriented design principles and modern programming language features to produce unit tests that are fun to write and easy to review. Code snippets will be written in C#.

## Problem description

The functionality in question here consists of parsing a section of an ASP.NET application's configuration defined in `appsettings*.json` files. These files are read by the ASP.NET framework after configuring the application with an instruction such as `builder.Configuration.AddJsonFile($"appsettings.json", optional: false, reloadOnChange: true)`.

The configuration can then be accessed in a variety of ways. In our case, parsing the configuration section leads to a higher-level object and, more importantly, one in a valid state. For this reason, the most suitable way is to use an `IConfiguration` object from the NuGet package `Microsoft.Extensions.Configuration.Abstractions`.

This object is not easily mockable for our unit tests, but Microsoft has thought of us: `ConfigurationBuilder` - from the NuGet package `Microsoft.Extensions.Configuration` - is a convenient way of creating an in-memory configuration from a key-value dictionary, especially useful for unit testing, as it avoids file I/O and external dependencies.

```csharp
Dictionary<string, string> configuration = ...;
var jsonConfiguration = new ConfigurationBuilder()
    .AddInMemoryCollection(configuration)
    .Build();
```

The disadvantage is that the tree structure is lost: the path of entries in the configuration is encoded in the key, and in an atypical format: each level is separated from its parent by the `':'` character. We'd like to build this configuration in a syntax that makes it easy to bridge the gap with the real configuration written in JSON, and that doesn't require knowledge of the subtleties of input path encoding.

The configuration to build here has the following JSON format:

```json
{
  "AwsSecretManager" : {
    "RegionName": "***",
    "SecretName": "****",
    "PartnerKeyName": {               // Optional. Use default value if not defined  
      "Login": "MyPartnerLogin",      // default value is "Login" 
      "Password": "MyPartnerPassword" // default value is "Password" 
    },  
    "AwsCredentials": { 
        "Source": "Basic",
        "AccessKey": "~~~~",
        "SecretKey": "~~~~~~"
    },
    // or 
    "AwsCredentials": { "Source": "Environment" }
  }
}
```

To do this, there's nothing like experimentation: we write the construction code for this configuration section, testing this syntax from both sides: on the one hand, does the usage meet our needs? On the other hand, is it easy to implement, while at the same time preventing us from writing buggy code? The right compromise comes after a few attempts. Let's look at the result.

## Syntax on the usage side

Let's start with an example:

```csharp
var configuration =
    Configuration.Section("AwsSecretManager",
        Configuration.Value("RegionName", "***"),
        Configuration.Value("SecretName", "****"),
        Configuration.Section("PartnerKeyName",
            Configuration.Value("Login", "MyPartnerLogin"),
            Configuration.Value("Password", "MyPartnerPassword")),
        Configuration.Section("AwsCredentials",
            Configuration.Value("Source", "Environment"))
    ).ToConfiguration();
```

This syntax is based on two helpers and one method:

- `Configuration.Value(key, value)` helper defines an entry in a section.
- `Configuration.Section(name, ..entries)` helper defines a section with the specified name, containing the given entries. The `entries` parameter is a `param` array.
- Both helpers use a short naming, eliminating verbs like `Get`, `Make`, `Create` to simplify the syntax without compromising the clarity.
- `ToConfiguration()` is used to define the `IConfiguration` required as input to unit tests.

This approach offers several key advantages:

- **No JSON Required:** Configuration is constructed directly in C#, eliminating the need to embed, parse, or comment JSON within your code. The resulting unit tests are more robust and have better performances.
- **Natural Nesting:** The syntax's structure closely matches the nesting found in JSON. Sections can contain other sections or entries, making the configuration tree intuitive and easy to follow.
- **Lightweight Syntax:** The syntax does not rely on the `new` keyword, braces `{}` or square brackets `[]`, resulting in concise and readable code.
- **Code Clarity:** The syntax is explicit: qualified use with `Configuration.` removes any doubt about the domain involved.

## Implementation design

Let's take a look at the implementation behind this syntax.

```csharp
public static class Configuration
{
    public record Entry(string Key, string Value) : IEntries
    {
        public IEnumerator<Entry> GetEnumerator()
        {
            yield return this;
        }
    }

    public interface IEntries : IEnumerable<Entry>
    {
        IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();

        IConfigurationRoot ToConfiguration() =>
            new ConfigurationBuilder()
                .AddInMemoryCollection(this.ToDictionary(x => x.Key, x => x.Value)!)
                .Build();
    }

    private class Entries : IEntries
    {
        private readonly IEnumerable<Entry> _entries;

        private Entries(IEnumerable<Entry> entries)
        {
            _entries = entries;
        }

        public IEnumerator<Entry> GetEnumerator() =>
            _entries.AsEnumerable().GetEnumerator();

        public static Entries Collect(Func<Entry, Entry> f, params IEntries[] entries) =>
            new(entries.SelectMany(x => x.Select(f)));
    }

    public static IEntries Section(string name, params IEntries[] children) =>
        Entries.Collect(entry => entry with { Key = $"{name}:{entry.Key}" }, children);

    public static IEntries Value(string key, string value) =>
        new Entry(key, value);
}
```

Here are the steps I followed to arrive at this design:

To encourage the qualified use of helpers, all the code is placed in a static class `Configuration`. This is similar to what we do in F# to gather helpers into a module, although we lack the `RequireQualifiedAccess` attribute to force qualification when using functions in the module.

`Configuration` will contain the two helpers `Value` and `Section` that are static methods. `Value` constructs an instance of an `Entry` record used to define a `Key, Value` entry in configuration. `Section` takes as input the name and content of the section, the content being entries and sub-sections. What should be the input and output parameter types?

The output type of `Value` and `Section` must be compatible with the input type of `Section`. So we need a common abstraction. The `ConfigurationBuilder` and its `AddInMemoryCollection` method allows us to consider a section as a simple collection of entries, hence the `IEntries` interface which extends `IEnumerable<Entry>`. A default implementation of `IEnumerable.GetEnumerator` is provided: derived classes need only to implement `IEnumerable<T>.GetEnumerator`.

We need a type implementing the `IEntries` interface. We could create a `Section` type, but we don't actually need one. As the section name can just be encoded in the entry keys, an `Entries` type is sufficient. This is a class that encapsulates an `IEnumerable<Entry>` in order to delegate the construction of the enumerator to it. This class can be private, hiding this implementation detail from `Configuration` users.

We have linked `Section` to `IEntries`. What can we do on the `Entry` side to link it to `IEntries`? The _Composite_ design pattern points the way. `Entry` can be seen as a collection of a single entry: the type can thus implement the `IEntries` interface, with a `GetEnumerator` based on a simple `yield return this;`.

At this stage, we still need to manage the contents of a section, i.e. the aggregation of all entries and the encoding of the section name in the entry keys. This second aspect is handled directly by the `Section` helper, while entry aggregation is delegated to a `Collect` function in the `Entries` class, to keep responsibilities clearly separated. `Collect` creates a new instance of `Entries` wrapping a call to `SelectMany` to flatten the set of specified entries, each entry being mapped using the specified function `f`.

## Design principles followed in the implementation

Key design aspects:

- **No Inheritance:** The design avoids base classes, to favor composition over inheritance. For example, the `Entries` class does not inherit from an existing collection, but wraps it and delegates to it the implementation of the enumeration of its elements. The use of default implementations in the `IEntries` interface is at the limit of this principle, as it could be seen as inheritance, but it offers the advantage to avoid duplication.
- **Default Interface Methods and No Duplication:** The use of default method implementations (such as `GetEnumerator`) in `IEntries` interface allows code sharing without inheritance. `ToConfiguration` could have been written as an extension method, but it's easier to centralize them all in one place.
- **Encapsulation:** Implementation details are hidden. Only the `IEntries` interface and the `Entry` record are exposed, while the concrete classe `Entries` remains private. This would allow us to change our internal model, to rely on other implementations, without breaking the calling code.
- **Immutability:** The objects involved in this design are all immutable, guaranteeing that they are supplied directly in a valid state that cannot be corrupted afterwards, in particular the key of an entry encoding the access path. The `entry` type relies on the C# 9 `record` to get the immutability for free.
- **Composite Pattern:** The `IEntries` interface embodies the _Composite_ design pattern. This gives us the flexibility to combine as many inputs as we like.

## Conclusion

This configuration builder provides a robust and flexible way to build configuration objects in C#, leveraging modern language features and design principles for clarity and maintainability.

The investment in creating such APIs pays dividends in test maintainability—an area that is often overlooked in many codebases but crucial for long-term project health.

Are you familiar with this type of design? If you haven't written one yourself, has this article motivated you to give it a try?