---
title: Applying object-oriented design principles - In-memory configuration builder
description: Applying object-oriented design principles - In-memory configuration builder
tags: #csharp #dotnet #oop
published: false
# published_at: 2025-04-02 06:43 +0000
---

TODO: write an introduction - indicate the config ins JSON in appsettings - problem of embedding the JSON in a string (no more intelleisesne) or in an external file (input of the unit tests far away) -> solution: use in memory config, write a builder to get a nice syntax

TODO: remove the code

```csharp
using System.Collections;
using Microsoft.Extensions.Configuration;

namespace Xxx.Tests.Configuration;

/// <summary>
/// Fluent API to build in-memory configuration, with a nesting structure matching the JSON structure of the JSON in the <c>appsettings.json</c> file.
/// </summary>
/// <example>
/// To build the configuration corresponding to the following JSON configuration...
/// <code>
/// {
///     "AwsSecretManager" : {
///         "RegionName": "eu-central-1",
///         "SecretName": "cnc/partner-auth",
///         "AwsCredentials": {
///             "Source": "Environment"
///         }
///     }
/// }
/// </code>
/// ...use the following C# code:
/// <code>
/// var configurationBuilder =
///     Configuration.Section("AwsSecretManager",
///         Configuration.Value("RegionName", "eu-central-1"),
///         Configuration.Value("SecretName", "cnc/partner-auth"),
///         Configuration.Section("AwsCredentials",
///             Configuration.Value("Source", "Environment"))
///     ).ToAwsSecretManagerConfigurationBuilder();
/// var awsSecretManagerConfiguration = configurationBuilder.Build();
/// </code>
/// </example>
public static class Configuration
{
    // ReSharper disable once MemberHidesStaticFromOuterClass
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

    public static IEntries Section(string name, params IEntries[] entriesSeries) =>
        Entries.Collect(entry => entry with { Key = $"{name}:{entry.Key}" }, entriesSeries);

    public static IEntries Value(string key, string value) =>
        new Entry(key, value);
}
```

## Usage

The builder functions provide a clean and intuitive way to build configuration objects that mirror JSON structure:

TODO: indicate the namespaces

```csharp
var configuration =
    Configuration.Section("AwsSecretManager",
        Configuration.Value("RegionName", "eu-central-1"),
        Configuration.Value("SecretName", "cnc/partner-auth"),
        Configuration.Section("AwsCredentials",
            Configuration.Value("Source", "Environment"))
    );
```

This approach offers several key advantages:

- **No JSON Required:** Configuration is constructed directly in C#, eliminating the need to embed, parse, or comment JSON within your code. This keeps configuration logic type-safe and refactor-friendly.
- **Natural Nesting:** The API's structure closely matches the nesting found in JSON. Sections can contain other sections or values, making the configuration tree intuitive and easy to follow.
- **Lightweight Syntax:** The API avoids the `new` keyword, similar to F#, resulting in concise and readable code. Configuration entries and sections are created via static methods, streamlining the syntax.
- **Code Clarity:** The syntax is short, expressive, and handy to use. It's worth investing time in writing such APIs to increase the maintainability of unit tests, which is often what's missing in codebases.

## Design Principles

The implementation demonstrates solid object-oriented design principles:

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
}
```

Key design aspects:

- **No Inheritance:** The design avoids base classes and inheritance, favoring composition. This keeps the API simple and focused.
- **Encapsulation:** Implementation details are hidden. Only the `IEntries` interface is exposed, while concrete classes remain internal, ensuring a clean separation between interface and implementation.
- **Default Interface Methods:** The use of default method implementations (such as `GetEnumerator`) in interfaces allows code sharing without inheritance. Extension methods could achieve similar results, but default interface methods provide a more direct approach.
- **Composite Pattern:** The `IEntries` interface embodies the Composite design pattern:
  - A single `Entry` implements `IEntries`, so "one is many."
  - Multiple `IEntries` can be combined into a single `IEntries` via the `Collect` method, enabling flexible composition of configuration trees.

## Conclusion

This configuration builder provides a robust, flexible, and idiomatic way to build configuration objects in C#, leveraging modern language features and design patterns for clarity and maintainability. The investment in creating such APIs pays dividends in test maintainability—an area that is often overlooked in many codebases but crucial for long-term project health.

In F#, this pattern could be further simplified using a discriminated union, allowing for even more concise and expressive configuration definitions. However, the C# implementation demonstrates how modern C# features can create elegant and functional-style APIs while maintaining the language's object-oriented strengths.
