# ProtoBuf.jl

[![Build Status](https://travis-ci.org/tanmaykm/ProtoBuf.jl.png)](https://travis-ci.org/tanmaykm/ProtoBuf.jl)
[![ProtoBuf](http://pkg.julialang.org/badges/ProtoBuf_0.3.svg)](http://pkg.julialang.org/?pkg=ProtoBuf&ver=0.3)
[![ProtoBuf](http://pkg.julialang.org/badges/ProtoBuf_0.4.svg)](http://pkg.julialang.org/?pkg=ProtoBuf&ver=0.4)
[![ProtoBuf](http://pkg.julialang.org/badges/ProtoBuf_0.5.svg)](http://pkg.julialang.org/?pkg=ProtoBuf)

[**Protocol buffers**](https://developers.google.com/protocol-buffers/docs/overview) are a language-neutral, platform-neutral, extensible way of serializing structured data for use in communications protocols, data storage, and more.

**ProtoBuf.jl** is a Julia implementation for protocol buffers.


## Getting Started

Reading and writing data structures using ProtoBuf is similar to serialization and deserialization. Methods `writeproto` and `readproto` can write and read Julia types from IO streams respectively.

````
julia> using ProtoBuf

julia> type MyType                          # here's a Julia composite type
         intval::Int
         strval::ASCIIString
         MyType() = new()
         MyType(i,s) = new(i,s)
       end

julia> iob = PipeBuffer();

julia> writeproto(iob, MyType(10, "hello world"));   # write an instance of it

julia> readproto(iob, MyType())  # read it back into another instance
MyType(10,"hello world")
````

## Protocol Buffer Metadata

ProtoBuf serialization can be customized for a type by defining a `meta` method on it. The `meta` method provides an instance of `ProtoMeta` that allows specification of mandatory fields, field numbers, and default values for fields for a type. Defining a specialized `meta` is done simply as below:

````
import ProtoBuf.meta

meta(t::Type{MyType}) = meta(t,                          # the type which this is for
		Symbol[:intval],                                 # required fields
		Int[8, 10],                                      # field numbers
		Dict{Symbol,Any}({:strval => "default value"}))  # default values
````

Without any specialized `meta` method:

- All fields are marked as optional (or repeating for arrays)
- Numeric fields have zero as default value
- String fields have `""` as default value
- Field numbers are assigned serially starting from 1, in the order of their declaration.

For the things where the default is what you need, just passing empty values would do. E.g., if you just want to specify the field numbers, this would do:

````
meta(t::Type{MyType}) = meta(t, [], [8,10], Dict())
````

## Setting and Getting Fields
Types used as protocol buffer structures are regular Julia types and the Julia syntax to set and get fields can be used on them. But with fields that are set as optional, it is quite likely that some of them may not have been present in the instance that was read. Similarly, fields that need to be sent need to be explicitly marked as being set. The following methods are exported to assist doing this:

- `get_field(obj::Any, fld::Symbol)` : Gets `obj.fld` if it has been set. Throws an error otherwise.
- `set_field!(obj::Any, fld::Symbol, val)` : Sets `obj.fld = val` and marks the field as being set. The value would be written on the wire when `obj` is serialized. Fields can also be set the regular way, but then they must be marked as being set using the `fillset` method.
- `add_field!(obj::Any, fld::Symbol, val)` : Adds an element with value `val` to a repeated field `fld`. Essentially appends `val` to the array `obj.fld`.
- `has_field(obj::Any, fld::Symbol)` : Checks whether field `fld` has been set in `obj`.
- `clear(obj::Any, fld::Symbol)` : Marks field `fld` of `obj` as unset.
- `clear(obj::Any)` : Marks all fields of `obj` as unset.

The `protobuild` method makes it easier to set large types with many fields:
- `protobuild{T}(::Type{T}, nvpairs::Dict{Symbol}()=Dict{Symbol,Any}())`

````
julia> using ProtoBuf

julia> type MyType                # here's a Julia composite type
           intval::Int
           MyType() = (a=new(); clear(a); a)
           MyType(i) = new(i)
       end

julia> type OptType               # and another one to contain it
           opt::MyType
           OptType() = (a=new(); clear(a); a)
           OptType(o) = new(o)
       end

julia> iob = PipeBuffer();

julia> writeproto(iob, OptType(MyType(10)));

julia> readval = readproto(iob, OptType());

julia> has_field(readval, :opt)       # valid this time
true

julia> writeproto(iob, OptType());

julia> readval = readproto(iob, OptType());

julia> has_field(readval, :opt)       # but not valid now
false
````

Note: The constructor for types generated by the `protoc` compiler have a call to `clear` to mark all fields of the object as unset to start with. A similar call must be made explicitly while using Julia types that are not generated. Otherwise any defined field in an instance is assumed to be valid.


The `isinitialized(obj::Any)` method checks whether all mandatory fields are set. It is useful to check objects using this method before sending them. Method `writeproto` results in an exception if this condition is violated.

````
julia> using ProtoBuf

julia> import ProtoBuf.meta

julia> type TestType
           val::Any
       end

julia> type TestFilled
           fld1::TestType
           fld2::TestType
           TestFilled() = (a=new(); clear(a); a)
       end

julia> meta(t::Type{TestFilled}) = meta(t, Symbol[:fld1], Int[], Dict{Symbol,Any}());

julia> tf = TestFilled()
TestFilled(#undef,#undef)

julia> isinitialized(tf)      # false, since fld1 is not set
false

julia> set_field!(tf, :fld1, TestType(""))

julia> isinitialized(tf)      # true, even though fld2 is not set yet
true
````

## Equality &amp; Hash Value
It is possible for fields marked as optional to be in an &quot;unset&quot; state. Even bits type fields (`isbits(T) == true`) can be in this state though they may have valid contents. Such fields should then not be compared for equality or used for computing hash values. All ProtoBuf compatible types must override `hash`, `isequal` and `==` methods to handle this. The following unexported utility methods can be used for this purpose:

- `protohash(v)` : hash method that considers fill status of types
- `protoeq{T}(v1::T, v2::T)` : equality method that considers fill status of types
- `protoisequal{T}(v1::T, v2::T)` : isequal method that considers fill status of types

The code generator already generates code for the types it generates overriding `hash`, `isequal` and `==` appropriately.

## Other Methods
- `copy!{T}(to::T, from::T)` : shallow copy of objects
- `isfilled(obj::Any, fld::Symbol)` : same as `has_field`
- `isfilled(obj::Any)` : same as `isinitialized`
- `fillset(obj::Any, fld::Symbol)` : mark field fld of object obj as set
- `fillunset(obj::Any)` : mark all fields of this object as not set
- `fillunset(obj::Any, fld::Symbol)` : mark field fld of object obj as not set
- `lookup(en::ProtoEnum,val::Integer)` : lookup the name (symbol) corresponding to an enum value
- `enumstr(enumname, enumvalue::Int32)`: returns a string with the enum field name matching the value


## Generating Code (from .proto files)
The Julia code generator plugs in to the `protoc` compiler. It is implemented as `ProtoBuf.Gen`, a sub-module of `ProtoBuf`. The callable program (as required by `protoc`) is provided as the script `ProtoBuf/plugin/protoc-gen-julia`.

To generate Julia code from `.proto` files, add the above mentioned `plugin` folder to the system `PATH` environment variable, so that `protoc` can find the `protoc-gen-julia` executable. Then invoke `protoc` with the `--julia_out` option. 

E.g. to generate Julia code from `proto/plugin.proto`, run the command below which will create a corresponding file `jlout/plugin.jl`.

`protoc -I=proto --julia_out=jlout proto/plugin.proto`

Each `.proto` file results in a corresponding `.jl` file, including one each for other included `.proto` files. Separate `.jl` files are generated with modules corresponding to each top level package.

If a field name in a message or enum matches a Julia keyword, it is prepended with an `_` character during code generation.

If a package contains a message which has the same name as the package itself, optionally set the `JULIA_PROTOBUF_MODULE_POSTFIX=1` environment variable when running `protoc`, this will append `_pb` to the module names.

### Julia Type Mapping

.proto Type | Julia Type        | Notes
---         | ---               | ---
double      | Float64           | 
float       | Float64           | 
int32       | Int32             | Uses variable-length encoding. Inefficient for encoding negative numbers – if your field is likely to have negative values, use sint32 instead.
int64       | Int64             | Uses variable-length encoding. Inefficient for encoding negative numbers – if your field is likely to have negative values, use sint64 instead.
uint32      | UInt32            | Uses variable-length encoding.
uint64      | UInt64            | Uses variable-length encoding.
sint32      | Int32             | Uses variable-length encoding. Signed int value. These more efficiently encode negative numbers than regular int32s.
sint64      | Int64             | Uses variable-length encoding. Signed int value. These more efficiently encode negative numbers than regular int64s.
fixed32     | UInt32            | Always four bytes. More efficient than uint32 if values are often greater than 2^28.
fixed64     | UInt64            | Always eight bytes. More efficient than uint64 if values are often greater than 2^56.
sfixed32    | Int32             | Always four bytes.
sfixed64    | Int64             | Always eight bytes.
bool        | Bool              | 
string      | ByteString        | A string must always contain UTF-8 encoded or 7-bit ASCII text.
bytes       | Array{UInt8,1}    | May contain any arbitrary sequence of bytes.

### Generic Services
The Julia code generator generates code for generic services if they are switched on for either C++ `(cc_generic_services)`, Python `(py_generic_services)` or Java `(java_generic_services)`.

To use generic services, users provide implementations of the RPC controller, RPC channel, and service methods.

The RPC Controller must be an implementation of `ProtoRpcController`. It is not currently used by the generated code except for passing it on to the RPC channel.

The RPC channel must implement `call_method(channel, method_descriptor, controller, request)` and return the response.

Service stubs are Julia types. Stubs can be constructed by passing an RPC channel to the constructor. For each service, two stubs are generated:
- <servicename>Stub: The asynchronous stub that takes a callback to invoke with the result on completion
- <servicename>BlockingStub: The blocking stub that returns the result on completion

## Note:

- Extensions are not supported yet.
- Groups are not supported. They are deprecated anyway.
- Enums are declared as `Int32` types in the generated code, but a separate Julia type is generated with fields same as the enum values which can be used for validation. The types representing enums extend from the abstract type `ProtoEnum` and the `lookup` method can be used to verify valid values.

