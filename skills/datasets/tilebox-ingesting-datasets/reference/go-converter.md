# Go Converter

Use Go only when the user explicitly requests it or the converter already lives in a Go project; otherwise use Python. Go converters use a dataset-specific generated protobuf type and ingest it directly through tilebox-go. Go does not currently provide Python's high-level semantic Assets compiler, so direct Asset profile optimization and wire encoding is an explicit low-level boundary.

## Generate The Dataset Type

After creating and inspecting the dataset schema, generate its concrete Go type:

```bash
tilebox dataset generate \
  --slug <dataset-slug> \
  --out ./internal/protogen \
  --package <protobuf.package> \
  --name <MessageName>
```

`--out`, `--package`, and `--name` are optional; follow project conventions. Commit imported generated code, regenerate it after compatible schema updates, and never edit it manually.

## Build The Generated Message

Use the generated dataset builder/type for the aggregate datapoint and `github.com/tilebox/tilebox-go/protogen/datasets/stac/v1` for well-known STAC messages.

Conceptual structure:

```go
datapoint := generatedv1.Scene_builder{
    Time:           timestamppb.New(acquisitionTime),
    Geometry:       geometry,
    StacId:         new(sourceID),
    Assets:         assets,
    Links:          links,
    Storage:        storage,
    Authentication: authentication,
    CloudCover:     new(cloudCover),
}.Build()
```

Use the exact generated builder fields and preserve explicit presence. Do not replace the generated type with dynamic descriptors or a generic map.

## Construct Semantic Values First

Even without a Go semantic compiler, separate source conversion from wire compilation:

```text
source record
    -> provider-specific normalized Go structs
    -> canonical semantic Assets/Bands/Links/registries
    -> datasets.stac.v1 compact protobuf messages
    -> generated dataset message
```

Keep source parsing and STAC 1.1 normalization independent of profile indices. This makes canonical behavior testable and limits low-level code to one compiler package/function.

## Direct Assets Construction Is An Escape Hatch

Read `asset-profile-wire-format.md` before building `datasets.stac.v1.Assets`. The Go compiler layer must:

- create complete access profiles;
- preserve exact primary and alternate href semantics;
- merge and validate Storage/Authentication registries;
- resolve inheritance and lift only semantically identical Band metadata;
- intern complete Band messages with explicit presence;
- assign and remap valid profile indices; and
- produce deterministic output with semantic round-trip fixtures.

Do not port private Python implementation code or treat its profile ordering as a contract. Implement the public wire invariants with deterministic ordering and isolate the compiler so a future Go semantic builder can replace it.

If the project cannot implement and test that boundary safely, use Python for conversion or pause until a Go semantic builder exists. Do not emit a partially optimized or guessed message.

## Ingest Through tilebox-go

Ingest the generated messages directly with tilebox-go:

```go
client := datasets.NewClient()
_, err := client.Datapoints.Ingest(ctx, collectionID, &datapoints, true)
if err != nil {
    return err
}
```

Group datapoints by the approved collection. Use `allowExisting=true` for normal retries of identical payloads; changed records follow the recipe's revision policy. The client uses `TILEBOX_API_KEY` by default and batches at most 8192 values per service request.

Do not call the Connect ingestion service directly, marshal aggregate messages manually, or send raw bytes when the public tilebox-go client accepts generated messages.

Test source normalization separately from compilation. Cover every wire invariant, exact semantic reconstruction, deterministic output independent of Go map iteration, generated type compatibility, and bounded query-back. Prefer semantic assertions over protobuf golden bytes.

Keep direct compilation behind one project-local boundary, for example:

```go
func CompileAssets([]SemanticAsset) (*stacv1.Assets, *stacv1.Storage, *stacv1.Authentication, error)
```

When tilebox-go gains a semantic Assets API, replace this compiler while retaining source-normalization and semantic fixtures.
