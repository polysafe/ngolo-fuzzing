Ngolo-fuzzing
======

Ngolo-fuzzing is a tool to generate automatically a fuzz target against a golang package.

Motivation
------

The purpose of ngolo-fuzzing is to automate the generation of a fuzz target, without human assistance.
It is not meant to generate an optimized fuzz target, but a complete one, in terms of code coverage.

Usage
------

Simply :
`./ngolo-fuzzing github.com/catenacyber/ngolo-fuzzing/duggy`

More completely :
`./ngolo-fuzzing -exlude Must,Expand regexp outdir`

Ngolo-fuzzing requires one argument : the name of the golang package against which to create the fuzz target.

Ngolo-fuzzing can have a second argument, a name of a directory where to output the results, default is `fuzz_ng`.

Ngolo-fuzzing has one argument `exclude` to exclude from fuzzing functions containing (as in `strings.Contains`) a list of patterns separated by commas.

Output
------

Ngolo-fuzzing will output two files in the output directory :
- A protobuf file named ngolofuzz.proto, describing the golang package API
- A golang file fuzz_ng.go containing the fuzz targets, ie the `Fuzz` functions

Compile and run the fuzzer
------

Ngolo-fuzzing simply generates the golang code for the fuzz target. It does not compile it.
To get a fuzzer running, you also need to run
```
# generate fuzz_ng.pb.go out of ngolofuzz.proto
protoc --go_out=./ ngolofuzz.proto
# Use go114-fuzz-build to build a static library
go114-fuzz-build -func FuzzNG_unsure -o fuzz_ng.a ./fuzz_ng
# link with libFuzzer
$CXX $CXXFLAGS $LIB_FUZZING_ENGINE fuzz_ng.a -o fuzz_ng
```

You can also use libprotobuf-mutator in the compiling scheme cf lpm/ngolofuzz.cc...

To get a debug output when you have a crash, you can run the fuzzer on the crash input with environment variable `FUZZ_NG_REPRODUCER` set to a file name to be written.
In this file, there will be written the list of functions called with their arguments.

Status
------

This is now a working Proof Of Concept.
It is working against many packages of the standard library but not all of them.
Even when it is working against a package (ie it produces a valid fuzz target), the fuzz target is not always complete in terms of coverage.
Warnings are printed out to show what is not covered, like usage of `io.ReaderWriterCloser` in an argument, or a function as an argument cf `ast.FuncType`.

Ngolo-fuzzing assumes that the golang package being fuzzed is not meant to panic with a list of calls of its functions.
This assumption is obviously wrong, cf `regexp.MustCompile`.
It is also wrong for functions not mentioning `panic` in its documentation like `regexp.Expand`.
Current workaround is to (manually) exclude these functions from the fuzz target with the `exclude` option of `ngolo-fuzzing`.

How does it work ?
------

**Why Golang ?**

Golang makes this easy with
  - Providing an `ast` package in its standard library to parse the golang package
  - Using a garbage collector, so the fuzz target does not need to care about releasing the resources it created.

Most of the ideas can be reused for other languages, but still need to code again much of this.

**Why Protobuf ?**

Protobuf allows to describe a golang package interface, by describing the list of functions with their arguments.
There may be other options, like go-fuzz-headers, that could work.

Protobuf unserialization, or libprotobuf-mutator, will generate a structure out of the fuzzing input as bytes.
This structure is a list of enum/union of each function, described by its arguments.

One function's argument can either be :
  - natively described by protobuf like `uint32`
  - can be generated out of a native types of protobuf like using `strings.NewReader` to get a `io.RuneReader` out of a `string` from protobuf
  - can be generated by the package, as a return of a function. The fuzz target stores these results for reuse.

TODOs
------

* Implement a tool to generate a corpus element out of a golang main program (make a protobuf out of parsing ast)
* Complete the print function get the complete program cf `FUZZ_NG_REPRODUCER`
* Add tests
* Complete duggy for testing
* Check all std library builds (like implement `io.ReadWriterCloser`)
* Implement a more focused version on only one function, building the necessary arguments for it through code generation if needed
* Builds corpus/dictionary out of unit tests: hook functions (especially ones with string or []byte args), output data, and convert it to protobuf format
