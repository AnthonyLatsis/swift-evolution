# Migration tooling for Swift upcoming features

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Anthony Latsis](https://github.com/AnthonyLatsis)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Implementation: TBD
* Review: TBD

## Introduction

As Swift evolves, source-breaking changes to the language are staged behind new
language modes and, more recently, [upcoming features][SE-0362] that may be
enabled piecemeal without adopting an entire new language mode.

In this proposal, we zoom in on the experience of adopting individual
upcoming features.
The premise is that Swift needs a new means to providing quality assistance
with code migration or adoption, and that, in principle, code migration can be
performed incrementally without blocking development or breaking source.

[SE-0362]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0362-piecemeal-future-features.md

## Motivation

Adjusting to new behaviors or language requirements may demand research,
coordinated efforts, careful consideration, and manual code refactorings,
sometimes on a case-by-case basis.
For many projects, source-breaking language features are expected to generate
hundreds of targeted errors.
It is also likely for developers to encounter knock-on collateral errors that
are not directly indicative of the exact cause or source of the problem.
We should strive for a better migration experience than blocking development on
atomic adoption and requiring programmers to slog through such a volume of
diagnostics.

Something about the following being good illustrations of the problems
described above:

* [SE-0337] Strict concurrency [TBD]

* [SE-0409]: Introduces the concept of access levels on import declarations to
  give library developers control over which module dependencies are exposed to
  clients and an upcoming feature for defaulting to internal imports to bias
  toward hiding dependencies rather than exposing them. This feature can break
  source across modules and may require a coordinated adoption effort in large
  projects.

* [SE-0444] Fixing member import visibility: The goal of this proposal is to reduce the likelihood of ambiguities arising from the use of extension members. Swift’s existing rules about when imports are required when referencing external declarations in a source file are inconsistent and they lead to extension members being visible even when the module they are declared in has not been imported. This proposal amends the rules for member lookup to make them consistent with top-level name lookup, giving developers more direct control over which extension members are visible.

* Our [prospective vision for improving the approachability of data-race safety][vision]
  explores various source-breaking changes that could be made in order to ease
  the transition to data-race safety and match developer expectations.
  some areas. For example, the Inherit isolation by default for async functions pitch proposes a source breaking change to improve the developer experience and the changes could be automated.

* [SE-0335] introduced syntax for existential or boxed types to visually
  distinguish them from conformance constraints and deliver contrast between
  dynamic existential types and fixed `some` types. The associated
  `ExistentialAny` feature is a vivid example where automatic migration is
  possible, but not desirable. The niche that exitential types occupy combined
  with their lighweight syntax make for an enticing and viable, yet often
  misused, abstraction construct. In places where type dynamism may be
  superfluous, we want to guide developers toward an informed choice of
  abstraction.

[vision]: https://github.com/hborla/swift-evolution/blob/approachable-concurrency-vision/visions/approachable-concurrency.md
[SE-0335]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0335-existential-any.md
[SE-0337]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0337-support-incremental-migration-to-concurrency-checking.md
[SE-0409]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0409-access-level-on-imports.md
[SE-0444]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0444-member-import-visibility.md

## Proposed Solution

We propose to introduce the notion of a source-compatible preview mode that,
when implemented for a given upcoming feature, can be enabled for guidance on
preserving the behavior of existing code or improving it under the feature.

## Detailed Design

* **Compiler**:

  On the compiler side, the `-enable-upcoming-feature` option will support an
  optional mode specifier with `preview` as the only valid mode:

  ```
  -enable-upcoming-feature <feature>[:mode]

  mode := "preview"
  ```

  Example:

  ```
  -enable-upcoming-feature InternalImportsByDefault:preview
  ```

* **Swift Package Manager**:

  The `PackageDescription` library will augment the
  `SwiftSetting.enableUpcomingFeature` method with an optional parameter for
  the mode and introduce a corresponding enumeration type.

  ```swift
  @available(_PackageDescription, introduced: 6.2)
  public enum SwiftFeatureMode {
    case preview
    case on
  }
  ```

  ```diff
      @available(_PackageDescription, introduced: 5.8)
      public static func enableUpcomingFeature(
          _ name: String,
  +       mode: SwiftFeatureMode = .on,
          _ condition: BuildSettingCondition? = nil
      ) -> SwiftSetting {
+         let argument = switch mode {
+             case .preview: "\(name):preview"
+             case .mode: name
+         }
+
          return SwiftSetting(
-             name: "enableUpcomingFeature", value: [name], condition: condition)
+             name: "enableUpcomingFeature", value: [argument], condition: condition)
      }
  ```

  Example:

  ```
  SwiftSetting.enableUpcomingFeature("InternalImportsByDefault", mode: .preview)
  ```

Preview mode does not apply to experimental features per this proposal.

Features enabled in preview mode shall not break source, that is, result in
errors or behavioral changes.
To avoid false impressions and confusion, the compiler will emit a warning and
consider the feature disabled if preview mode is requested for a known feature
that does not support it.

The objective of preview mode is to offer comprehensive assistance with
code migration and/or adoption in the shape of warnings, notes, remarks, and
fix-its — as and when appropriate.
Preview mode cannot promise to offer source-compatible assistance because source
impact is generally nondeterministic.
Neither can it promise to always offer fix-its for the same reason in regards
to user intention.

### Diagnostics

> NB: It will generally be hard for developers to work out diagnostic affiliation,
> especially when multiple features are enabled in preview mode. We may want to
> vend the diagnostic group (named after the feature) to IDEs and annotate
> diagnostics accordingly (`-print-diagnostics-groups`) in our formatters.

## Source compatibility

This proposal has no impact on existing code.

## ABI compatibility

This proposal has no impact on existing code.

## Implications on adoption

## Future directions

TBD. Talk about `swift migrate`.

## Alternatives considered

?

## Acknowledgements

This proposal is inspired by and builds on drafts devised by Allan Shortlidge
(@tshortli) and Holly Borla (@bhorla).

