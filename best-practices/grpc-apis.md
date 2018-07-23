# Evolving gRPC APIs &mdash; Best Practices

This document covers rules and best practices for evolving gRPC APIs to allow
safe cross-version usage.

## High Level Guidance

The most important takeaway from this document should be that changing protocol
buffers used across software releases requires careful planning and analysis of
behavior both in future releases and past releases. Every change to a protobuf
defintion should include an analysis of how future releases will cope with the
field if it is not present as well as how past releases will behave given that
they will be blind to the new field.

## Rules

### R1. Never, _EVER_ Re-use Protcol Buffer Field Numbers

Every protocol buffer field has an identifying number. When
[encoded](https://developers.google.com/protocol-buffers/docs/encoding) on the
wire, this number and the field's type are the only information that is actually
passed. The result is that it is _never_ safe to re-use a field number. Protocol
buffers provides the `reserved` keyword allowing field numbers to be marked as
reserved and prevent them from being re-used. Any field number that has been
used as part of publicly available code (not _just_ releases; this rule applies
to anything checked into an official repo) should be considered "spent" and
marked as reserved if the field is later removed.

```proto
reserved 1, 3;

string name = 2;

int32 size = 4;  // <-- The first available field number is 4.
```

### R2. Avoid Introducing and Enable a Feature in the same Release

Maintaining downgrade compatibility when launching a new release requires that
the release not include any features not supported by the previous release. In
the strictest sense this means that new features cannot be "enabled" until one
release after they are deemed "working". As a concrete example, when Google
Compute Engine introduced multi-queue networking we did so one release later
than when it was "fully supported" in GCE production. This allowed us to safely
roll-back the release when a bug elsewhere was encountered while not bricking
VMs that had booted after multi-queue had been enabled.

TODO: Find a concrete example from Kata rather than from an external source.

### R3. Enums are Problematic

Related to R2, enum types are problematic as adding additional values in a new
releases can create undefined behavior in prior releases.

### R4. Rarely Change Field Types

Related to the R1, while not always strictly unsafe it is usually unwise to
change the type of an existing field. In most cases it is better to inroduce a
new field with the new type and slowly phase out the old field once it has left
the backward compatibility window.

Exceptions to this rule can include converting enum types to int32 or int64,
provided no values other than the original enum definition are used

## Guidelines

### GL1. Have Clearly Defined Compatibility Windows

TODO: Change this guideline to encompass Kata's agreed-upon compatibiltiy
policy.

It is critical to have a clear policy on what upgrade and downgrade paths are
supported. One policy that has been generally successful for other large
projects (Google Compute Engine, in particular) it to support unbounded forward
upgrades and single-release downgrades.

### GL2. The Scope of Compatibility is Only Releases that may Mix

Many compatibility problems can been addressed be avoiding mixing components.
For example, if each Kata release includes both the agent and the runtime, the
agent can be

### GL3. Redundancy is a Useful Tool

While it can be aesthetically displeasing, redundantly encoding similar (or
identical) information can allow slow migration from one behavior

### GL4. Changes to Defaults Should Straddle Compatibility Boundaries

As per R2 above, new fields cannot generally be introduced and used in the same
release. More broadly, new behaviors (typically new features) should not become
the default until the _oldest_ release not supporting that behavior is no longer
in the downgrade compatibiltiy window.

### GL5. On-the-Wire Versioning is Rarely Useful

Very few protocol buffers at Google include version information as part of the
wire format (either explicitly or implicitly). For example, the Google Compute
Engine API is still on "version 1" despite dozens of calls and arguments being
added since its inception. This is possible because new features are rolled out
as "additions" to existing APIs, and by following R2 above.

While it can seem attractive to add explicit versions to the protocol this
typically requires complex version conversion infrastructure that must be
maintained (and tested) alongside the actual code.
