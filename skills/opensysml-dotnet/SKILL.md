---
name: opensysml-dotnet
description: Use the .NET client `OpenMBEE.OpenSysML` (.NET 10, a `ProjectReference` to the OpenSysML checkout, not on NuGet) to drive OpenSysML from C# — open a private `sysml-grpc` or connect to a shared one over gRPC or Connect, load models, read diagnostics and symbols with their members, read an element's metamodel properties, evaluate typed values, instantiate and list every path to an object, verify, execute, model-check, run repeatedly, convert, query, render and edit, with an `OpenSysMLException` family that keeps refused calls apart from reported model failures. Also generates typed C# classes over a model and reads exported JSON into metamodel types without a service. Use when writing C# that reads, runs or checks SysML v2 models.
license: Apache-2.0
metadata:
  project: OpenSysML
  upstream: https://github.com/Open-MBEE/OpenSysML
---

# OpenSysML from C# and .NET

The client is built from the checkout and consumed by `ProjectReference`; it is not on NuGet:

```bash
git clone https://github.com/Open-MBEE/OpenSysML && cd OpenSysML
make build-grpc                      # bin/sysml-grpc, for tests and private connections
make dotnet-build                    # restore from the lock files and build with warnings as errors
```

```xml
<ItemGroup>
  <ProjectReference Include="path/to/OpenSysML/client/dotnet/src/OpenMBEE.OpenSysML/OpenMBEE.OpenSysML.csproj" />
</ItemGroup>
```

It targets `net10.0` only. Build it with the SDK the repository-root `global.json` pins, or a later
patch in the same feature band, and run `dotnet` from the repository root so that file selects it
(`make dotnet-preflight` checks both). There is no synchronous API, no Native AOT or trimming
support and no browser host.

In nightly builds and the release after v0.12.1. Verified against a build of OpenSysML `develop`
(`sysml v0.12.1-1013-gbcaed37df7`) with the client at `client/dotnet` of the same checkout
(`make dotnet-test` green). The client releases in lockstep with the core: a checkout of a release
tag downloads that release's `sysml-grpc`.

```csharp
using OpenMBEE.OpenSysML;

await using var connection = await Connection.OpenAsync();   // a private sysml-grpc child, stopped on dispose
var model = await connection.LoadAsync("model.sysml");        // the service reads the path
foreach (var d in model.Errors)                               // Diagnostic: Severity, Code, Message, Span
{
    Console.Error.WriteLine($"{d.Code}: {d.Message}");
}
model.RequireNoErrors();                                      // ModelErrorsException carrying them
var mass = await model.EvalAsync("mass", new EvalOptions { SubjectSymbolId = "Demo::sedan" });
Console.WriteLine(mass);                                      // 1800.0
```

A model with syntax errors still parses; branch on `d.Code` (`"syntax"`, `"unresolved"`, …), never on
message text. `LoadContentAsync(source)` parses inline notation;
`new ParseOptions { StrictConformance = true }` makes OpenSysML-only notation an error.
`LoadAsync` reads one document, so imports across files do not resolve: use
`ParseSourcesAsync([SourceDocument.FromFile("a.sysml"), SourceDocument.FromFile("b.sysml")])`.

## Connections

- `Connection.OpenAsync()` reaches `ConnectionOptions.Address`, else `$OPENSYSML_SERVICE`, else
  starts a **private child** (`sysml-grpc -port 0 -health-port 0 -report-address -exit-with-parent`)
  that exits with this process however it ends. A service named by address is never started or
  stopped by the client.
- Connections of one copy of the client share the private child while they agree on the binary and
  timeouts; the last one disposed stops it. `IsolatedService = true` gives a connection its own.
- `ConnectionOptions`: `Address`, `Protocol` (`Grpc` default, `Connect`), `Encoding` (`Protobuf`
  default, `Json`, Connect only), `Token` (else `$OPENSYSML_GRPC_TOKEN`), `RequestTimeout` (60 s),
  `Headers`, `MaxRequestSize`/`MaxResponseSize` (512 MiB), `Logger`, `Version` (exact, else
  `StaleServiceException`), `RequireCapabilities` (checked at open), `AutoStart`, `BinaryPath`,
  `ExpectedBinarySha256`, `IsolatedService`, `StartupTimeout`/`ShutdownTimeout` (30 s/5 s),
  `DownloadVersion`, `GithubRepo`. Every option is checked before anything is sent.
- The private child's binary: `BinaryPath`, `$OPENSYSML_GRPC_BINARY`, the shared cache
  `~/.opensysml/bin/sysml-grpc` (under the lock every client shares), downloading the release the
  client was built with (or `DownloadVersion`, `$OPENSYSML_GRPC_VERSION`) into it, then `sysml-grpc`
  on `PATH`. A download is trusted only against the SHA-256 pins the checkout embeds; the client does
  not verify a release's Sigstore signature, so an unpinned release is `UnpinnedReleaseException`
  before anything is fetched unless `OPENSYSML_ALLOW_UNPINNED_DOWNLOAD` grants same-origin trust. An
  unreleased checkout pins nothing for itself: set `OPENSYSML_GRPC_BINARY` to a `make build-grpc`
  binary (`opensysml-install`).

```csharp
await using var debugging = await Connection.OpenAsync(new ConnectionOptions
{
    Protocol = Protocol.Connect,
    Encoding = MessageEncoding.Json,              // readable traffic; protobuf is the default
    IsolatedService = true,
    RequireCapabilities = [Capabilities.Query],   // CapabilityException at open, not at first use
});
Console.WriteLine(debugging.ServerInfo);          // sysml-grpc <version>, <count> capabilities
bool converts = debugging.Capabilities.Has(Capabilities.Convert);
```

Decide on capability names, never on the version. Every typed call checks the capabilities its
arguments need and throws `CapabilityException` (`Capability`, `Remedy`) before sending; a field a
service fills only under a capability reads as empty or null from one without it.

## `Model` operations

Each method is one RPC and takes its own options class, so a call accepts only what its RPC uses:

| Area | Methods | Result |
|---|---|---|
| Symbols | `SymbolAsync`, `FindSymbolAsync`, `FindAsync`, `LookupAsync` | `Symbol`; the `Find` forms answer null for an unknown name, the others throw `SymbolNotFoundException` (`Suggestions`) |
| Element properties | `ElementPropertiesAsync` | `ElementProperties` |
| Diagnostics | `GetDiagnosticsAsync` | `ValueList<Diagnostic>` under a lint selection |
| Expressions | `EvalAsync`, `EvaluateAsync` | `Value`; `Evaluation` keeps its diagnostics |
| Objects | `InstantiateAsync`, `InstancePathsAsync`, `InstancePathsAtAsync` | `Instantiation`, `InstancePaths` |
| Behavior | `ExecuteActionAsync`, `ExecuteStateAsync`, `ExploreActionAsync`, `ExploreStateAsync`, `CheckActionAsync`, `CheckStateAsync` | `ActionRun`, `StateRun`, `Exploration`, `CheckResult` |
| Verification | `VerifyConstraintAsync`, `VerifyRequirementAsync`, `VerifySatisfactionAsync`, `SatisfiedAsync`, `ValidateInstanceAsync` | `Verification`, `Satisfaction`, `bool`, `Validation` |
| Calculation and analysis | `CalcAsync`, `RunAnalysisAsync`, `ExploreAnalysisAsync`, `RunSweepAsync` | `Calculation`, `Analysis`, `Exploration`, `Sweep` |
| Repeated runs | `RunActionRunsAsync`, `RunAnalysisRunsAsync`, `SelfCheckAsync`, `RecordRunAsync` | `Runs`, `SelfCheck`, `Recording` |
| Conversion | `ConvertAsync`, `ToNotationAsync`, `ToTurtleAsync`, `ToApiJsonAsync`, `SaveAsync` | `Conversion` |
| Queries and documents | `QueryAsync`, `QueryOslcAsync`, `RunDocumentQueryAsync`, `RenderDocumentAsync`, `RenderViewAsync` | `QueryResult`, `DocumentQueryResult`, `RenderedDocument`, `RenderedView` |
| Editing | `Edit()`, `ApplyEditsAsync` | an `Editor` collecting one atomic batch; `EditResult` |
| Graphs and data | `ExportGraphsAsync`, `ImportDataAsync`, `CompareResultsAsync` | `Graphs`, `ImportResult`, `Comparison` |

`connection.ConvertAsync`, `MigrateAsync` (SysML v1) and `ListEnginesAsync` need no model. Nothing is
written to disk until a `SaveAsync`/`WriteAsync`, which writes the whole file through a temporary one
and never overwrites unless `SaveOptions.Overwrite` asks.

```csharp
var built = await model.InstantiateAsync("Demo::sedan");
var builtMass = built.Root.FeatureValues["mass"].Value;                                           // 1800.0

var run = await model.ExecuteActionAsync("Demo::addFive",
    new Dictionary<string, Value> { ["x"] = new Value.IntegerValue(10) });
var result = run.Outputs["result"];                                                               // 15, for out result : Integer[1]

var v = await model.VerifyRequirementAsync("Demo::Vehicle::lightEnough",
    new VerifyOptions { SubjectSymbolId = "Demo::truck" });
Console.WriteLine($"{v.Holds} {v.Verdict.Decided} {v.Verdict.Condition}");                         // False True mass < 2000.0

var json = await model.ToApiJsonAsync();                                                          // json.Content; json.SaveAsync(path)
```

An output or calc result declared without `[1]` answers a `SequenceValue` of its values; declare `[1]`
to read the scalar. A false verdict, a failed sweep row or a failed run among repeated runs is an
answer, never an exception.

## Members, element properties and object paths

```csharp
var sedan = await model.SymbolAsync("Demo::sedan");
_ = sedan.Members;                 // own then inherited: SymbolMember(Id, Name, Kind, IsInherited, InheritedFrom, IsLibrary)
_ = sedan.InheritedMembers;        // InheritedFrom names the declaring type
_ = sedan.WithheldLibraryMembers;  // standard-library members counted, not listed, until:
var withLibrary = await model.SymbolAsync("Demo::sedan", new SymbolOptions { IncludeLibraryMembers = true });

var read = await model.ElementPropertiesAsync("Demo::Vehicle", ["inheritedFeature", "isAbstract", "connectorEnd"]);
_ = read.Find("inheritedFeature")?.Elements;   // ElementRefs; their Id is accepted back
_ = read.Find("isAbstract")?.Value;            // an ElementPropertyValue arm, a BooleanValue here
_ = read.Find("connectorEnd")?.Unsupported;    // why there are no values: "PartDefinition declares no property connectorEnd"

var engine = built.Reachable[1];
_ = engine.ReachedBy;              // InstanceReach: InstanceId, FeatureName, Index, Path; null for the root
_ = engine.Path;                   // "engine"
_ = built.FindOwner(engine);       // the holding object
var paths = await model.InstancePathsAsync(engine.Id);   // every root-relative InstancePath, aliases included
```

`Members` is empty from a service without `symbol_members` (`ChildIds` still names the owned ones);
`IncludeLibraryMembers`, `ElementPropertiesAsync` and `InstancePathsAsync`/`InstancePathsAtAsync`
throw `CapabilityException` before sending without `symbol_members`, `element_properties` or
`instance_paths`. Omit the property names to read every property the element carries.
`TypeFacts.Types` are a feature's derived types (`Feature::type`), beside the typing as written.

## Values

`Value` is a sealed hierarchy of records — `IntegerValue`, `BigIntegerValue`, `RationalValue`,
`RealValue`, `NumberValue`, `ComplexValue`, `BooleanValue`, `TextValue`, `QuantityValue`,
`VectorValue`, `VectorQuantityValue`, `TensorQuantityValue`, `MeasurementRefValue`,
`EnumLiteralValue`, `InstanceRefValue`, `MetaobjectValue`, `FunctionValue`, `SequenceValue`,
`SetValue`, `ArrayValue`, `InfinityValue`, `NullValue`, `UnsetValue`, `UndeterminedValue` — so switch on
it with a default arm:

```csharp
string Describe(Value value) => value switch
{
    Value.RealValue real => $"{real.Value} (Real)",
    Value.IntegerValue integer => $"{integer.Value} (Integer)",
    Value.UnsetValue or Value.UndeterminedValue => "no value",   // answers, never 0 or false
    _ => value.ToString(),
};
Console.WriteLine(Describe(mass));                                // 1800 (Real)
```

Equality is by content, not SysML's numeric comparison: `IntegerValue(1)` is not `RealValue(1.0)`.

## Exceptions

Everything derives from `OpenSysMLException`, except `OperationCanceledException` and
`ObjectDisposedException`. A model with diagnostics, a false verdict or a failed run is a result; an
exception means the call could not be made or was refused:

- **Refused before sending:** `InvalidRequestException`, `CapabilityException`.
- **Service start and binaries:** `ServiceStartException` and its `BinaryNotFoundException`
  (`SearchedLocations`), `ChecksumMismatchException`, `UnpinnedReleaseException`,
  `UnsupportedPlatformException`; `StaleServiceException` for a `Version` mismatch.
- **Transport and status:** `TransportException`; `ServiceException` (`StatusCode`, `ServiceMessage`,
  `Method`) and its `AuthenticationException`, `ModelNotFoundException`, `ModelFileNotFoundException`.
- **Answered with a model failure:** `ModelException` (`Diagnostics`) and its `ModelErrorsException`,
  `SymbolNotFoundException`, `ExecutionException`, `CheckException`, `VerificationException`
  (`Reason`) and its `AnalysisException` (`Partial`), `QueryException`, `EditException`,
  `ConversionException`, `MigrationException`.

```csharp
try
{
    await model.LookupAsync("Demo::sedann");
}
catch (SymbolNotFoundException missing)
{
    Console.Error.WriteLine($"{missing.Name}? did you mean {string.Join(", ", missing.Suggestions)}");
}
```

## Typed classes, metamodel types and raw calls

- **Typed model classes.** A console project of the checkout writes C# interfaces and classes over a
  model's definitions, committed beside your code; `--check` fails when they are stale:
  `make dotnet-generate SOURCES='model.sysml' NAMESPACE=Acme.Model OUTPUT=ModelTypes.cs [CHECK=1]`.
  `Vehicle.FromInstance(await model.InstantiateAsync("Demo::sedan"))` then reads an instance as
  `IVehicle`.
- **Exported JSON without a service.** Reference `client/dotnet/src/OpenMBEE.OpenSysML.Metamodel`
  alone and call `MetamodelGraph.ReadJson(path)` on an `api-json` export (`sysml -convert api-json`); a
  property the export omits throws `NotSuppliedException`, never a default.
- **Raw calls.** `connection.CallAsync<TRequest, TResponse>("GetServerInfo", request)` reaches any RPC
  with the generated `OpenMBEE.OpenSysML.Wire` messages; it checks no capability and interprets no
  result.

```csharp
var raw = await connection.CallAsync<OpenMBEE.OpenSysML.Wire.ServerInfoRequest, OpenMBEE.OpenSysML.Wire.ServerInfoResponse>(
    "GetServerInfo", new OpenMBEE.OpenSysML.Wire.ServerInfoRequest());
Console.WriteLine(raw.Capabilities.Count > 0);                    // True
```

## Pitfalls

- Reuse one `Connection` and dispose it with `await using`; `OpenSysMLClient.LoadAsync` opens a
  connection the returned model owns.
- Use qualified names (`Package::Def::feature`) wherever a symbol id is asked for.
- The client never retries a call, falls back to another protocol or bypasses TLS validation.
- It does not decode the `QueryOutcome` detail of a refused query: read the `ServiceException`'s
  message, or run `sysml -run-query … -json` (`opensysml-qvts`).
- Write models with `opensysml-sysml-authoring`; `sysml -validate` reports the same diagnostics as
  `model.ParseDiagnostics`. `client/dotnet/README.md`, `docs/clients/dotnet.md` and
  `docs/reference/dotnet-api.md` in the OpenSysML repository are the full reference.
