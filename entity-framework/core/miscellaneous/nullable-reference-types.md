---
title: Working with nullable reference types - EF Core
author: roji
ms.date: 9/9/2019
ms.assetid: bde4e0ee-fba3-4813-a849-27049323d301
uid: core/miscellaneous/nullable-reference-types
---
# Working with Nullable Reference Types

C# 8 introduced a new feature called [nullable reference types](/dotnet/csharp/tutorials/nullable-reference-types), allowing reference types to be annotated, indicating whether it is valid for them to contain null or not. If you are new to this feature, it is recommended that make yourself familiar with it by reading the C# docs.

This page introduces EF Core's support fo nullable reference types, and describes best practices for working with them.

## Required and optional properties

The main documentation on required and optional properties and their interaction with nullable reference types is the [Required and Optional Properties](xref:core/modeling/required-optional) page. It is recommended you start out by reading that page first.

> [!NOTE]
> Exercise caution when enabling nullable reference types on an existing project: reference type properties which were previously configured as optional will now be configured as required, unless they are explicitly annotated to be nullable. When managing a relational database schema, this may cause migrations to be generated which alter the database column's nullability.

## DbContext and DbSet

When nullable reference types are enabled, the C# compiler emits warnings for any uninitialized non-nullable property, as these would contain null. As a result, the common practice of defining a non-nullable `DbSet` on a context will now generate a warning. However, EF Core always initializes all `DbSet` properties on DbContext-derived types, so they are guaranteed to never be null, even if the compiler is unaware of this. Therefore, it is recommended to keep your `DbSet` properties non-nullable - allowing you to access them without null checks - and to silence the compiler warnings by explicitly setting them to null with the help of the null-forgiving operator (!):

[!code-csharp[Main](../../../samples/core/Miscellaneous/NullableReferenceTypes/NullableReferenceTypesContext.cs?name=Context&highlight=3-4)]

## Non-nullable properties and initialization

Compiler warnings for uninitialized non-nullable reference types are also a problem for regular properties on your entity types. In our example above, we avoided these warnings by using [constructor binding](xref:core/modeling/constructors), a feature which works perfectly with non-nullable properties, ensuring they are always initialized. However, in some scenarios constructor binding isn't an option: navigation properties, for example, cannot be initialized in this way.

Required navigation properties present an additional difficulty: although a dependent will always exist for a given principal, it may or may not be loaded by a particular query, depending on the needs at that point in the program ([see the different patterns for loading data](xref:core/querying/related-data)). At the same time, it is undesirable to make these properties nullable, since that would force all access to them to check for null, even if they are required.

One way to deal with these scenarios, is to have a non-nullable property with a nullable [backing field](xref:core/modeling/backing-field):

[!code-csharp[Main](../../../samples/core/Miscellaneous/NullableReferenceTypes/Order.cs?range=12-17)]

Since the navigation property is non-nullable, a required navigation is configured; and as long as the navigation is properly loaded, the dependent will be accessible via the property. If, however, the property is accessed without first properly loading the related entity, an InvalidOperationException is thrown, since the API contract has been used incorrectly.

As a terser alternative, it is possible to simply initialize the property to null with the help of the null-forgiving operator (!):

[!code-csharp[Main](../../../samples/core/Miscellaneous/NullableReferenceTypes/Order.cs?range=19)]

An actual null value will never be observed except as a result of a programming bug, e.g. accessing the navigation property without properly loading the related entity beforehand.

> [!NOTE]
> Collection navigations, which contain references to multiple related entities, should always be non-nullable. An empty collection means that no related entities exist, but the list itself should never be null.

## Navigating and including nullable relationships

When dealing with optional relationships, it's possible to encounter compiler warnings where an actual null reference exception would be impossible. When translating and executing your LINQ queries, EF Core guarantees that if an optional related entity does not exist, any navigation to it will simply be ignored, rather than throwing. However, the compiler is unaware of this EF Core guarantee, and produces warnings as if the LINQ query were executed in memory, with LINQ to Objects. As a result, it is necessary to use the null-forgiving operator (!) to inform the compiler that an actual null value isn't possible:

[!code-csharp[Main](../../../samples/core/Miscellaneous/NullableReferenceTypes/Program.cs?range=46)]

A similar issue occurs when including multiple levels of relationships across optional navigations:

[!code-csharp[Main](../../../samples/core/Miscellaneous/NullableReferenceTypes/Program.cs?range=36-39&highlight=2)]

If you find yourself doing this a lot, and the entity types in question are predominantly (or exclusively) used in EF Core queries, consider making the navigation properties non-nullable, and to configure them as optional via the Fluent API or Data Annotations. This will remove all compiler warnings while keeping the relationship optional; however, if your entities are traversed outside of EF Core, you may observe null values although the properties are annotated as non-nullable.

## Scaffolding

[The C# 8 nullable reference type feature](/dotnet/csharp/tutorials/nullable-reference-types) is currently unsupported in reverse engineering: EF Core always generates C# code that assumes the feature is off. For example, nullable text columns will be scaffolded as a property with type `string` , not `string?`, with either the Fluent API or Data Annotations used to configure whether a property is required or not. You can edit the scaffolded code and replace these with C# nullability annotations. Scaffolding support for nullable reference types is tracked by issue [#15520](https://github.com/aspnet/EntityFrameworkCore/issues/15520).
