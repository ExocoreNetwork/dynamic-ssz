# Dynamic SSZ (dynssz)

Dynamic SSZ (`dynssz`) is a Go library designed to provide flexible and dynamic SSZ encoding/decoding for Ethereum data structures. It stands out by using runtime reflection to handle serialization and deserialization of types with variable field sizes, enabling it to support a wide range of Ethereum presets beyond the mainnet. `dynssz` integrates with `fastssz` to leverage static type information for encoding/decoding when possible, but its primary advantage lies in its ability to adapt to dynamic field sizes that are not well-suited to static code generation methods.

`dynssz` is designed to bridge the gap between the efficiency of static SSZ encoding/decoding and the flexibility required for handling dynamic data structures. It achieves this through a hybrid approach that combines the best of both worlds: leveraging `fastssz` for static types and dynamically processing types with variable sizes.

## Benefits

- **Flexibility**: Supports Ethereum data structures beyond mainnet presets, accommodating custom and dynamic specifications.
- **Hybrid Efficiency**: Balances the efficiency of static processing with the flexibility of dynamic handling, optimizing performance where possible.
- **Developer-Friendly**: Simplifies the handling of SSZ data for developers by abstracting the complexity of dynamic data processing.

## Installation

To install `dynssz`, use the `go get` command:

```shell
go get github.com/pk910/dynamic-ssz
```

This will download and install the `dynssz` package into your Go workspace.

## Usage

### Struct Tag Annotations for Dynamic Encoding/Decoding

`dynssz` utilizes struct tag annotations to indicate how fields should be encoded/decoded, supporting both static and dynamic field sizes:

- `ssz-size`:
Defines static default field sizes. This tag follows the same format supported by `fastssz`, allowing seamless integration.
- `dynssz-size`:
Specifies dynamic sizes based on specification properties, extending the flexibility of `dynssz` to adapt to various Ethereum presets. Unlike the straightforward `ssz-size`, `dynssz-size` supports not only direct references to specification values but also simple mathematical expressions. This feature allows for dynamic calculation of field sizes based on spec values, enhancing the dynamic capabilities of `dynssz`.

    The `dynssz-size` tag can interpret and evaluate expressions involving one or multiple spec values, offering a versatile approach to defining dynamic sizes. For example:
    
    - A direct reference to a single spec value might look like `dynssz-size:"SPEC_VALUE"`.
    - A simple mathematical expression based on a spec value could be `dynssz-size:"(SPEC_VALUE*2)-5"`, enabling the size to be dynamically adjusted according to the spec value.
    - For more complex scenarios involving multiple spec values, the tag can handle expressions like `dynssz-size:"(SPEC_VALUE1*SPEC_VALUE2)+SPEC_VALUE3"`, providing a powerful tool for defining sizes that depend on multiple dynamic specifications.

    When processing a field with a `dynssz-size` tag, `dynssz` evaluates the expression to determine the actual size. If the resolved size deviates from the default established by `ssz-size`, the library switches to dynamic handling for that field. This mechanism ensures that `dynssz` can accurately and efficiently encode or decode data structures, taking into account the intricate sizing requirements dictated by dynamic Ethereum presets.

Fields with static sizes do not need the `dynssz-size` tag. Here's an example of a structure using both tags:

```go
type BeaconState struct {
    GenesisTime                  uint64
    GenesisValidatorsRoot        phase0.Root `ssz-size:"32"`
    Slot                         phase0.Slot
    Fork                         *phase0.Fork
    LatestBlockHeader            *phase0.BeaconBlockHeader
    BlockRoots                   []phase0.Root `ssz-size:"8192,32" dynssz-size:"SLOTS_PER_HISTORICAL_ROOT,32"`
    StateRoots                   []phase0.Root `ssz-size:"8192,32" dynssz-size:"SLOTS_PER_HISTORICAL_ROOT,32"`
    ...
}
```

### Creating a New DynSsz Instance

```go
import "github.com/pk910/dynamic-ssz"

// Define your dynamic specifications
specs := map[string]any{
    "SYNC_COMMITTEE_SIZE": uint64(32),
    // ...
}

ds := dynssz.NewDynSsz(specs)
```

### Marshaling an Object

```go
data, err := ds.MarshalSSZ(myObject)
if err != nil {
    log.Fatalf("Failed to marshal SSZ: %v", err)
}
```

### Unmarshaling an Object

```go
err := ds.UnmarshalSSZ(&myObject, data)
if err != nil {
    log.Fatalf("Failed to unmarshal SSZ: %v", err)
}
```

## Performance

The performance of `dynssz` has been benchmarked against `fastssz` using BeaconBlocks and BeaconStates from small kurtosis testnets, providing a consistent and comparable set of data. These benchmarks compare three scenarios: exclusively using `fastssz`, exclusively using `dynssz`, and a combined approach where `dynssz` defaults to `fastssz` for static types that do not require dynamic processing. The results highlight the balance between flexibility and speed:

**Legend:**
- First number: Unmarshalling time in milliseconds.
- Second number: Marshalling time in milliseconds.

### Mainnet Preset

#### BeaconBlock Decode + Encode (10,000 times)
- **fastssz only:** [4 ms / 2 ms] success
- **dynssz only:** [356 ms / 422 ms] success
- **dynssz + fastssz:** [12 ms / 6 ms] success

#### BeaconState Decode + Encode (10,000 times)
- **fastssz only:** [12416 ms / 7817 ms] success
- **dynssz only:** [38020 ms / 25964 ms] success
- **dynssz + fastssz:** [11256 ms / 8135 ms] success

### Minimal Preset

#### BeaconBlock Decode + Encode (10,000 times)
- **fastssz only:** [0 ms / 0 ms] failed (unmarshal error)
- **dynssz only:** [347 ms / 582 ms] success
- **dynssz + fastssz:** [251 ms / 283 ms] success

#### BeaconState Decode + Encode (10,000 times)
- **fastssz only:** [0 ms / 0 ms] failed (unmarshal error)
- **dynssz only:** [8450 ms / 8036 ms] success
- **dynssz + fastssz:** [1554 ms / 1096 ms] success

These results showcase the dynamic processing capabilities of `dynssz`, particularly its ability to handle data structures that `fastssz` cannot process due to its static nature. While `dynssz` introduces additional processing time, its flexibility allows it to successfully manage both mainnet and minimal presets. The combined `dynssz` and `fastssz` approach significantly improves performance while maintaining this flexibility, making it a viable solution for applications requiring dynamic SSZ processing.

## Internal Technical Overview

### Key Components

- **Type and Value Size Calculation**: The library distinguishes between type sizes (static sizes of types or -1 for dynamic types) and value sizes (the absolute size of an instance in SSZ representation), utilizing recursive functions to accurately determine these sizes based on reflection and tag annotations (`ssz-size`, `dynssz-size`).

- **Encoding/Decoding Dispatch**: Central to the library's architecture are the `marshalType` and `unmarshalType` functions. These serve as entry points to the encoding and decoding processes, respectively, dynamically dispatching tasks to specialized functions based on the nature of the data (e.g., `marshalStruct`, `unmarshalArray`).

- **Dynamic Handling with Static Efficiency**: For types that do not necessitate dynamic processing (neither the type nor its nested types have dynamic specifications), `dynssz` optimizes performance by invoking corresponding `fastssz` functions. This ensures minimal overhead for types compatible with static processing.

- **Size Hints and Spec Values**: `dynssz` intelligently handles sizes through `sszSizeHint` structures, derived from field tag annotations. These hints inform the library whether to process data statically or dynamically, allowing for precise and efficient data serialization.

### Architecture Flow

1. **Size Calculation**: Upon receiving a data structure for encoding or decoding, `dynssz` first calculates its size. For encoding, it determines whether the structure can be processed statically or requires dynamic handling. For decoding, it assesses the expected size of the incoming SSZ data.

2. **Dynamic vs. Static Path Selection**: Based on the size calculation and the presence of dynamic specifications, the library selects the appropriate processing path. Static paths leverage `fastssz` for efficiency, while dynamic paths use runtime reflection.

3. **Recursive Encoding/Decoding**: The library recursively processes each field or element of the data structure. It dynamically navigates through nested structures, applying the correct encoding or decoding method based on the data type and size characteristics.

4. **Specialized Function Dispatch**: For complex types (e.g., slices, arrays, structs), `dynssz` dispatches tasks to specialized functions tailored to handle specific encoding or decoding needs, ensuring accurate and efficient processing.


## Contributing

We welcome contributions from the community! Please check out the [CONTRIBUTING.md](CONTRIBUTING.md) file for guidelines on how to contribute to `dynssz`.

## License

`dynssz` is licensed under the [LGPL License](LICENSE). See the LICENSE file for more details.

## Acknowledgements

Thanks to all the contributors and the Ethereum community for providing the inspiration and foundation for this project.
