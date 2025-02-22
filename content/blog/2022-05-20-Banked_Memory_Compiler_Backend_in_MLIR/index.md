+++
title = "Banked Memory Compiler Backend in MLIR"
[extra]
bio = """
  [Andrew Butt](https://andrewb1999.github.io/) is a first-year PhD student at Cornell's ECE department;
  [Nikita Lazarev](https://www.nikita.tech/) is a third-year PhD student at Cornell's ECE department.
"""
[[extra.authors]]
name = "Andrew Butt"
link = "https://andrewb1999.github.io/"  # Links are optional.
[[extra.authors]]
name = "Nikita Lazarev"
link = "https://www.nikita.tech/"  # Links are optional.
+++

### Code

Unfortunately, as this work is related to ongoing research, we are not yet ready to open-source the project. We have shared the code with Adrian which is available [here](https://www.github.com). If anyone else in the class in interested in seeing the code please reach out and we will give you access as well.

### Introduction

Banked memories, or logical memories that are divided into multiple physical memories, are an important part of achieving high performance FPGA designs. Unfortunately, neither RTL- nor HLS-based languages provide sufficient abstractions to express banked memories. This results in tremendous efforts that programmers need to put when designing memory sub-systems for their hardware architectures. The problem is especially hard given the diverse set of applications with different memory access patterns and performance requirements, as well as numerous hardware primitives the memory can be map onto (e.g., LUTRAM, BRAM, ULTRARAM, etc.) available in today's FPGAs. Without these abstractions, designers have to manually plan the microarchitecture of the memories with all the necessary optimizations in order to meet the target design requirements.

To address the aforementioned challenges and simplify the process of designing banked memories, we propose AMC -- a novel MLIR dialect built on top of CIRCT and Calyx that enables compilation of arbitrary memory structures. Our contribution lies in the following three aspects:

* the AMC dialect -- a programming IR abstraction for expressing banked memories;
* the AMC compiler -- a reference implementation of the AMC dialect built on Calyx;
* an optimization pass showcasing how our dialect can enable implementation of efficient memory structures when lowering down to hardware.


### The AMC dialect

We build the AMC dialect on top of Calyx. Calyx is an IR for compilers generating hardware accelerators. The banked memory specification in the AMC dialect involves four parts: (1) interface specification, (2) memory allocation specification, (3) port specification, and (4) port-interface mapping.

*The memory interface* specifies all read and write interfaces to a unit of banked memory with the corresponding set of arguments. The arguments include the address space size, data width, read/write semantics, and memory access latency in cycles. Example of an interface definition is shown in the listing bellow. Here we define a memory component *@test1* that implements a banked memory with four ports (two on read, two on write), each having 512 32-bit words and 1 cycle of guaranteed access latency.
```mlir
amc.memory @test1(!amc.port<512xf32, r, 1>, !amc.port<512xf32, w, 1>, !amc.port<512xf32, r, 1>, !amc.port<512xf32, w, 1>);
```

*The memory allocation description* specifies how the memory is divided into banks. For example, ```%0 = amc.alloc : !amc.memref<1024xf32, bank [2]>``` says that the memory contains two equivalent banks with 1024 32-bit words in total, and the result is assigned to SSA value \%0.

*The port specification* assigns read/write ports to the memory banks described earlier. For example,
```mlir
%3 = amc.create_port(%0 : !amc.memref<1024xf32, bank [2]>) banks [1] : !amc.port<512xf32, r, 1>
```
describes a read port connected to the bank 2 of the memory referred by \%0. The `amc.port` type has three argument attributes, the geometry of the port, the read/write semantics, and the port access latency in cycles.

Finally, *port-interface mapping* specifies how exactly the bank ports are mapped onto the memory interface ports defined earlier. A completed example of possible banked memory definition with our AMC dialect is shown in the listing bellow:
```mlir
amc.memory @test1(!amc.port<512xf32, r, 1>, !amc.port<512xf32, w, 1>, !amc.port<512xf32, r, 1>, !amc.port<512xf32, w, 1>) {
  // Specify memory allocation.
  %0 = amc.alloc : !amc.memref<1024xf32, bank [2]>

  // Specify ports.
  %1 = amc.create_port(%0 : !amc.memref<1024xf32, bank [2]>) banks [0] : !amc.port<512xf32, r, 1>
  %2 = amc.create_port(%0 : !amc.memref<1024xf32, bank [2]>) banks [0] : !amc.port<512xf32, w, 1>
  %3 = amc.create_port(%0 : !amc.memref<1024xf32, bank [2]>) banks [1] : !amc.port<512xf32, r, 1>
  %4 = amc.create_port(%0 : !amc.memref<1024xf32, bank [2]>) banks [1] : !amc.port<512xf32, w, 1>
 
  // Specify I/O mapping.
  amc.extern %1, %2, %3, %4 : !amc.port<512xf32, r, 1>, !amc.port<512xf32, w, 1>, !amc.port<512xf32, r, 1>, !amc.port<512xf32, w, 1>
}
```

The `amc.extern` operation, used in the example above, maps internal SSA values to the top level ports specified in the interface. This can be thought of similar to a return operator, where internal operations are made accesible to some external user. All top level ports specified in the interface must be mapped to an internal port via the `amc.extern` operator. There is no such thing as an "input" port to the module at this level of the representation.

### Lowering

The goal of lowering is to eventually convert the specification above into hardware. There already exists a lowering method from Calyx to Verilog, so all we need to do is lower the specification to Calyx. The general lowering pipeline can be view as follows:

Handshake Lowering -> Merge Lowering -> Bank Lowering -> Optimization -> AmcToCalyx

Our general goal here is to perform the straight forward lowering passes first, and then optimize later one we have fully expanded the memory structure. We will dive into each of these lowering passes below.

### Bank Lowering

We will start with the simplest lowering step. This should be viewed as the final lowering pass pre-optimization and simply translates a banked memref into multiple unbanked memref. This step requires that all create ports only access a single bank and create only latency-sensitive ports.

Here is an example of bank lowering:
```mlir
// Input
amc.memory @test1(!amc.port<512xf32, r, 1>, !amc.port<512xf32, w, 1>, !amc.port<512xf32, r, 1>, !amc.port<512xf32,
  %0 = amc.alloc : !amc.memref<1024xf32, bank [2]>
  %1 = amc.create_port(%0 : !amc.memref<1024xf32, bank [2]>) banks [0] : !amc.port<512xf32, r, 1>
  %2 = amc.create_port(%0 : !amc.memref<1024xf32, bank [2]>) banks [0] : !amc.port<512xf32, w, 1>
  %3 = amc.create_port(%0 : !amc.memref<1024xf32, bank [2]>) banks [1] : !amc.port<512xf32, r, 1>
  %4 = amc.create_port(%0 : !amc.memref<1024xf32, bank [2]>) banks [1] : !amc.port<512xf32, w, 1>
   
  amc.extern %1, %2, %3, %4 : !amc.port<512xf32, r, 1>, !amc.port<512xf32, w, 1>, !amc.port<512xf32, r, 1>, !amc.p
}

// Output after bank lowering
amc.memory @test1(!amc.port<512xf32, r, 1>, !amc.port<512xf32, w, 1>, !amc.port<512xf32, r, 1>, !amc.port<512xf32, w, 1>) {
  %0 = amc.alloc : !amc.memref<512xf32>
  %1 = amc.alloc : !amc.memref<512xf32>
  %2 = amc.create_port(%0 : !amc.memref<512xf32>) : !amc.port<512xf32, r, 1>
  %3 = amc.create_port(%0 : !amc.memref<512xf32>) : !amc.port<512xf32, w, 1>
  %4 = amc.create_port(%1 : !amc.memref<512xf32>) : !amc.port<512xf32, r, 1>
  %5 = amc.create_port(%1 : !amc.memref<512xf32>) : !amc.port<512xf32, w, 1>
  amc.extern %2, %3, %4, %5 : !amc.port<512xf32, r, 1>, !amc.port<512xf32, w, 1>, !amc.port<512xf32, r, 1>, !amc.port<512xf32, w, 1>
}
```
We can see that the original `amc.memref` is split into two smaller memrefs. This corresponds to lowering the logical banks of the original memory to physical, distinct banks. As the original memory contains 1024 elements with a bank factor of 2, the output contains two 512 element memories (representing each of the original 2 banks). The `amc.create_port` operators are also retargeted to the new memories. `create_port` operators that accessed bank 0 in the input now take memref 0 as input and operators that accessed bank 1 in the input now take memref 1 as input. These are semantically equivalent, but the output provides a representation closer to the implementation in hardware.

### Merge Lowering

The purpose of merge lowering is to translate latency-sensitive port creations that access more than one bank on a memory into multiple ports that access individual banks. These smaller ports are then combined with the merge operator to produce a port of the original size. This is analogous to how a port of this type would actually be implemented in hardware and allows future lowering and optimization passes to have a better idea of how many ports are actually accessing each bank.

The merge operator is a new primitive that takes multiple latency-sensitive ports and produces one larger latency-sensitive port. This can be thought of as a type of mux, which based on the top `ceil(log_2(n))` bits of the address will select one of the `n` ports to forward the access to.

Here is an example of merge lowering:
```mlir
// Input
amc.memory @test2(!amc.port<512xf32, rw, 1>, !amc.port<512xf32, r, 1>, !amc.port<1024xf32, w, 1>) {   
  %0 = amc.alloc : !amc.memref<1024xf32, bank [2]>   
  %1 = amc.create_port(%0 : !amc.memref<1024xf32, bank [2]>) banks [0] : !amc.port<512xf32, r, 1>   
  %2 = amc.create_port(%0 : !amc.memref<1024xf32, bank [2]>) banks [1] : !amc.port<512xf32, r, 1>   
  %3 = amc.create_port(%0 : !amc.memref<1024xf32, bank [2]>) banks [0, 1] : !amc.port<1024xf32, w, 1>   

  amc.extern %1, %2, %3 : !amc.port<512xf32, r, 1>, !amc.port<512xf32, r, 1>, !amc.port<1024xf32, w, 1>   
}

// Output after merge lowering
amc.memory @test2(!amc.port<512xf32, rw, 1>, !amc.port<512xf32, r, 1>, !amc.port<1024xf32, w, 1>) {
  %0 = amc.alloc : !amc.memref<1024xf32, bank [2]>
  %1 = amc.create_port(%0 : !amc.memref<1024xf32, [2]>) banks [0] : !amc.port<512xf32, r, 1>
  %2 = amc.create_port(%0 : !amc.memref<1024xf32, [2]>) banks [1] : !amc.port<512xf32, r, 1>
  %3 = amc.create_port(%0 : !amc.memref<1024xf32, [2]>) banks [0] : !amc.port<512xf32, w, 1>
  %4 = amc.create_port(%0 : !amc.memref<1024xf32, [2]>) banks [1] : !amc.port<512xf32, w, 1>
  %5 = amc.merge(%3, %4 : !amc.port<512xf32, w, 1>, !amc.port<512xf32, w, 1>) : !amc.port<1024xf32, w, 1>
  amc.extern %1, %2, %5 : !amc.port<512xf32, r, 1>, !amc.port<512xf32, r, 1>, !amc.port<1024xf32, w, 1>
}

// Output after merge lowering and bank lowering
amc.memory @test2(!amc.port<512xf32, rw, 1>, !amc.port<512xf32, r, 1>, !amc.port<1024xf32, w, 1>) {
  %0 = amc.alloc : !amc.memref<512xf32>
  %1 = amc.alloc : !amc.memref<512xf32>
  %2 = amc.create_port(%0 : !amc.memref<512xf32>) : !amc.port<512xf32, r, 1>
  %3 = amc.create_port(%1 : !amc.memref<512xf32>) : !amc.port<512xf32, r, 1>
  %4 = amc.create_port(%0 : !amc.memref<512xf32>) : !amc.port<512xf32, w, 1>
  %5 = amc.create_port(%1 : !amc.memref<512xf32>) : !amc.port<512xf32, w, 1>
  %6 = amc.merge(%4, %5 : !amc.port<512xf32, w, 1>, !amc.port<512xf32, w, 1>) : !amc.port<1024xf32, w, 1>
  amc.extern %2, %3, %6 : !amc.port<512xf32, r, 1>, !amc.port<512xf32, r, 1>, !amc.port<1024xf32, w, 1>
}
```

Comparing the input to the output after merge lowering, we can see that the write port (SSA value \%3 in the input) has been lowered to two smaller ports that are merged together. The two read ports are not lowered because they already access only a single bank. The newly created write ports in the output are only 512 elements instead of the original 1024 elements, but are then merged back together into a 1024 element port. This lowered stucture is again closer to how a port that accesses multiple banks is realized in hardware. We can also see that the `amc.extern` types have not changed and the port produced by the merge is now the one externalized.

### Handshake Lowering

Throughout the design of the AMC dialect, we discovered that it is important to distinguish between latency-sensitive and latency-insensitive (handshake) ports. Unlike latency-sensitive ports that provide a fixed latency between a read request and data availability, handshake ports must wait for a valid signal before data is available. Handshake ports are useful for irregular paralleism, where the access patterns are not known at compile time. This allows for memories that can provide a consistent latency in most cases, but stall one or more accesses in the case of a bank conflict.

Hanshake ports must be lowered before being fed into the rest of the lowering pipeline. Currently, we lower handshake ports in the most naive way possible, assuming that a future optimization pass will combine arbiters in a sensible way. This pass has not yet been implemented.

To support handshake ports we must introduce a new primitive, the arbiter. The job of an arbiter is to take in some number of latency-senstive port ssa value and produce some number of latency-insensitive handshake port SSA values. These handshake port ssa values can then be passed with amc.extern to the top level IO. At this level, the job of the arbiter is to prevent multiple handshake ports from trying to access the same latency-sensitive port at the same time. We will later have to choose a specific arbitration scheme (fixed-priority, round-robin, etc.) to meet the needs to the memory interface.

Here is an example of handshake lowering:
```mlir
// Input
amc.memory @test2(!amc.port_hs<1024xi32, rw>, !amc.port_hs<1024xi32, rw>) {
  %0 = amc.alloc : !amc.memref<1024xi32, bank [2]>
  %1 = amc.create_port(%0 : !amc.memref<1024xi32, [2]>) banks [0, 1] : !amc.port_hs<1024xi32, rw>
  %2 = amc.create_port(%0 : !amc.memref<1024xi32, [2]>) banks [0, 1] : !amc.port_hs<1024xi32, rw>
  amc.extern %1, %2 : !amc.port_hs<1024xi32, rw>, !amc.port_hs<1024xi32, rw>
}

// Output after only handshake lowering
amc.memory @test2(!amc.port_hs<1024xi32, rw>, !amc.port_hs<1024xi32, rw>) {
  %0 = amc.alloc : !amc.memref<1024xi32, bank [2]>
  %1 = amc.create_port(%0 : !amc.memref<1024xi32, [2]>) banks [0, 1] : !amc.port<1024xi32, rw, 1>
  %2 = amc.arbiter(%1 : !amc.port<1024xi32, rw, 1>) banks [0, 1] : !amc.port_hs<1024xi32, rw>
  %3 = amc.create_port(%0 : !amc.memref<1024xi32, [2]>) banks [0, 1] : !amc.port<1024xi32, rw, 1>
  %4 = amc.arbiter(%3 : !amc.port<1024xi32, rw, 1>) banks [0, 1] : !amc.port_hs<1024xi32, rw>
  amc.extern %2, %4 : !amc.port_hs<1024xi32, rw>, !amc.port_hs<1024xi32, rw>
}
```

Here we can see that each handshake port is lowered to a `create_port` that creates a latency-sensitive port and an arbiter. The arbiter in this case is just acting as a converter between handshake and latency-sensitive port, taking the latency-sensitive port as input and producing a handshake port. We can also see that the `amc.extern` still maintains the same type signatures, lining up with the top level handshake ports.

This example also shows what is meant by lowering in the most naive way. Each handshake port has its own arbiter which functionally just ties the valid signal high as each handshake port can always access the underlying latency-sensitive port. A future pass could co-optimize the underlying memory primitives and the handshake ports by merging together some arbiters to reduce the number of underlying ports.

We can then continue lowering the rest of the way:

```mlir
// Output after handshake lowering and merge lowering
amc.memory @test2(!amc.port_hs<1024xi32, rw>, !amc.port_hs<1024xi32, rw>) {
  %0 = amc.alloc : !amc.memref<1024xi32, bank [2]>
  %1 = amc.create_port(%0 : !amc.memref<1024xi32, [2]>) banks [0] : !amc.port<512xi32, rw, 1>
  %2 = amc.create_port(%0 : !amc.memref<1024xi32, [2]>) banks [1] : !amc.port<512xi32, rw, 1>
  %3 = amc.merge(%1, %2 : !amc.port<512xi32, rw, 1>, !amc.port<512xi32, rw, 1>) : !amc.port<1024xi32, rw, 1>
  %4 = amc.arbiter(%3 : !amc.port<1024xi32, rw, 1>) banks [0, 1] : !amc.port_hs<1024xi32, rw>
  %5 = amc.create_port(%0 : !amc.memref<1024xi32, [2]>) banks [0] : !amc.port<512xi32, rw, 1>
  %6 = amc.create_port(%0 : !amc.memref<1024xi32, [2]>) banks [1] : !amc.port<512xi32, rw, 1>
  %7 = amc.merge(%5, %6 : !amc.port<512xi32, rw, 1>, !amc.port<512xi32, rw, 1>) : !amc.port<1024xi32, rw, 1>
  %8 = amc.arbiter(%7 : !amc.port<1024xi32, rw, 1>) banks [0, 1] : !amc.port_hs<1024xi32, rw>
  amc.extern %4, %8 : !amc.port_hs<1024xi32, rw>, !amc.port_hs<1024xi32, rw>
}

// Output after handshake lowering, merge lowering, and bank lowering
amc.memory @test2(!amc.port_hs<1024xi32, rw>, !amc.port_hs<1024xi32, rw>) {
  %0 = amc.alloc : !amc.memref<512xi32>
  %1 = amc.alloc : !amc.memref<512xi32>
  %2 = amc.create_port(%0 : !amc.memref<512xi32>) : !amc.port<512xi32, rw, 1>
  %3 = amc.create_port(%1 : !amc.memref<512xi32>) : !amc.port<512xi32, rw, 1>
  %4 = amc.merge(%2, %3 : !amc.port<512xi32, rw, 1>, !amc.port<512xi32, rw, 1>) : !amc.port<1024xi32, rw, 1>
  %5 = amc.arbiter(%4 : !amc.port<1024xi32, rw, 1>) banks [0, 1] : !amc.port_hs<1024xi32, rw>
  %6 = amc.create_port(%0 : !amc.memref<512xi32>) : !amc.port<512xi32, rw, 1>
  %7 = amc.create_port(%1 : !amc.memref<512xi32>) : !amc.port<512xi32, rw, 1>
  %8 = amc.merge(%6, %7 : !amc.port<512xi32, rw, 1>, !amc.port<512xi32, rw, 1>) : !amc.port<1024xi32, rw, 1>
  %9 = amc.arbiter(%8 : !amc.port<1024xi32, rw, 1>) banks [0, 1] : !amc.port_hs<1024xi32, rw>
  amc.extern %5, %9 : !amc.port_hs<1024xi32, rw>, !amc.port_hs<1024xi32, rw>
}
```

### Optimization Pass: Memory Aggregation

After handshake, merge, and bank lowering we have a consistent way to optimize the underlying memory banks. To demonstrate optimization potentials, we implemented s memory aggregation pass. This pass allows to reduce the memory depth in favor of the word length. For example, a memory of type ```!amc.memref<1024xi32>``` can be converted to ```!amc.memref<512xi64>```. This optimization can be useful when it helps to better pack memory in the available hardware units. In particular, if the FPGAs ultra-RAM units have the width of 72 bits, it might be beneficial to aggregate memory as we showed in the above example.

The aggregation pass consists of four steps. The pass first replaces the type of memory allocation with another memory type of reduced depth. It then fixes all references to that memory for the `create_port` operations for all ports that use this memory. Then the pass injects a new operation `split_aggregate` that transforms ports of aggregated types into the ports of the original types as shown in the example bellow:
```mlir
%1 = amc.create_port(%0 : !amc.memref<256xi64>) : !amc.port<256xi64, w, 1>
%2 = amc.split_aggregated(%1 : !amc.port<256xi64, w, 1>) : !amc.port<512xf32, w, 1>
```

`split_aggregate` needs to be synthesized into a hardware circuit that implements the following functionality. It takes `N - 1` LSB of the original N-wide address and looks-up memory by this address. The resulting read/write word has the double width of the data port as in the memory specification. Then depending on the `N'th` LSB of the original address, either the LSB of the MSB part of the accessed word is getting assigned to the I/O ports.

The listing bellow shows example of the banked memory specification when applying the optimization.

```mlir
 // Original specification.
 amc.memory @test1(!amc.port<512xf32, r, 1>, !amc.port<512xf32, w, 1>, !amc.port<512xf32, r, 1>, !amc.port<512xf32, w, 1>) {
   %0 = amc.alloc : !amc.memref<1024xf32, bank [2]>
   %1 = amc.create_port(%0 : !amc.memref<1024xf32, bank [2]>) banks [0] : ! >
   %2 = amc.create_port(%0 : !amc.memref<1024xf32, bank [2]>) banks [0] : !amc.port<512xf32, w, 1>
   %3 = amc.create_port(%0 : !amc.memref<1024xf32, bank [2]>) banks [1] : !amc.port<512xf32, r, 1>
   %4 = amc.create_port(%0 : !amc.memref<1024xf32, bank [2]>) banks [1] : !amc.port<512xf32, w, 1>
   amc.extern %1, %2, %3, %4 : !amc.port<512xf32, r, 1>, !amc.port<512xf32, w, 1>, !amc.port<512xf32, r, 1>, !amc.port<512xf32, w, 1>
}

// Lowered and optimized specification.
amc.memory @test1(!amc.port<512xf32, r, 1>, !amc.port<512xf32, w, 1>, !amc.port<512xf32, r, 1>, !amc.port<512xf32, w, 1>) {
  %0 = amc.alloc : !amc.memref<256xi64>
  %1 = amc.alloc : !amc.memref<256xi64>
  %2 = amc.create_port(%0 : !amc.memref<256xi64>) : !amc.port<256xi64, r, 1>
  %3 = amc.split_aggregated(%2 : !amc.port<256xi64, r, 1>) : !amc.port<512xf32, r, 1>
  %4 = amc.create_port(%0 : !amc.memref<256xi64>) : !amc.port<256xi64, w, 1>
  %5 = amc.split_aggregated(%4 : !amc.port<256xi64, w, 1>) : !amc.port<512xf32, w, 1>
  %6 = amc.create_port(%1 : !amc.memref<256xi64>) : !amc.port<256xi64, r, 1>
  %7 = amc.split_aggregated(%6 : !amc.port<256xi64, r, 1>) : !amc.port<512xf32, r, 1>
  %8 = amc.create_port(%1 : !amc.memref<256xi64>) : !amc.port<256xi64, w, 1>
  %9 = amc.split_aggregated(%8 : !amc.port<256xi64, w, 1>) : !amc.port<512xf32, w, 1>
  amc.extern %3, %5, %7, %9 : !amc.port<512xf32, r, 1>, !amc.port<512xf32, w, 1>, !amc.port<512xf32, r, 1>, !amc.port<512xf32, w, 1>
}
```

### Lowering AMC Dialect down to Calyx

Lowering AMC to Calyx is currently the least developed part of the lowering flow, for a number of reasons. Lowering to Calyx turned out to be much more challenging than expected, especially when handling merge and arbitrate operations. Directly lowering merge and arbitrate operations to Calyx requires generators that can generate optimized Calyx components for each possible version of these primitives (address size, bit width, number of input and output ports, etc.). For these reasons, we are considering a more abstract intermediary language that arbitrate and merge can be lowered to before Calyx to simplify the process. This would also be helpful when supporting new primitives such as `split_aggregated`.

Calyx and the Calyx dialect also need some modifications to support a wider range of memories that can be generated by AMC. For example, Calyx does not currently support multiported memories or memories that generate verilog with a specific memory primtive (BRAM, UltraRAM, etc.). In theory, these primtives could be added through a Calyx library, but this is also currently unsupported by the Calyx MLIR Dialect.

As a result, the lowering from AMC to Calyx only support the amc.alloc and amc.create_port operations. All ports also must be single latency, read/write, and latency-sensitive. Here is an example lowering from AMC to Calyx:
```mlir
// Input to AMC to Calyx lowering
amc.memory @test1(!amc.port<512xf32, rw, 1>, !amc.port<512xf32, rw, 1>) {
  %0 = amc.alloc : !amc.memref<512xf32>
  %1 = amc.alloc : !amc.memref<512xf32>
  %2 = amc.create_port(%0 : !amc.memref<512xf32>) : !amc.port<512xf32, rw, 1>
  %3 = amc.create_port(%1 : !amc.memref<512xf32>) : !amc.port<512xf32, rw, 1>
  amc.extern %2, %3 : !amc.port<512xf32, rw, 1>, !amc.port<512xf32, rw, 1>
}

// Output
calyx.program "test1" {
  calyx.component @test1(%port_0.addr0: i9, %port_0.write_data: i32, %port_0.write_en: i1, %port_1.addr0: i9, %port_1.write_data: i32, %port_1.write_en: i1, %clk: i1 {clk}, %reset: i1 {reset}, %go: i1 {go}) -> (%port_0.read_data: i32, %port_1.read_data: i32, %done: i1 {done}) {
    %bank0.addr0, %bank0.write_data, %bank0.write_en, %bank0.clk, %bank0.read_data, %bank0.done = calyx.memory @bank0 <[512] x 32> [9] : i9, i32, i1, i1, i32, i1
    %bank1.addr0, %bank1.write_data, %bank1.write_en, %bank1.clk, %bank1.read_data, %bank1.done = calyx.memory @bank1 <[512] x 32> [9] : i9, i32, i1, i1, i32, i1
    calyx.wires {
      calyx.assign %bank0.clk = %clk : i1
      calyx.assign %bank1.clk = %clk : i1
      calyx.assign %bank0.write_data = %port_0.write_data : i32
      calyx.assign %bank0.write_en = %port_0.write_en : i1
      calyx.assign %port_0.read_data = %bank0.read_data : i32
      calyx.assign %bank0.addr0 = %port_0.addr0 : i9
      calyx.assign %bank1.write_data = %port_1.write_data : i32
      calyx.assign %bank1.write_en = %port_1.write_en : i1
      calyx.assign %port_1.read_data = %bank1.read_data : i32
      calyx.assign %bank1.addr0 = %port_1.addr0 : i9
    }
    calyx.control {
    }
  }
}
```

After lowering to Calyx, a final RTL design can be generated either through the native Calyx compiler or the experimental Calyx lowering within the CIRCT project.

### Challenges

The majority of the time spent in this project was on designing the IR to represent a wide range of banked memories. We considered a large number of sketched out examples ranging from simple array partitions to caches and data buffers that are way out of the scope of this project. Overall, we are happy with our decision to spend time on the IR itself and believe the resulting design can be extended in future research. Specifically, the separation between memories and ports along with latency-sensitive and latency-insensitive we believe are good decisions that expose optimization opportunities.

As mentioned above, the lowering from AMC to Calyx was also much more challenging than expected. Improving the robustness of this lowering pass will be one of the first steps before continuing research in other areas.

### Evaluation

Evaluation was another area that ended up being more challenging than expected. We have a flow to generate some subset of banked memories in Vitis HLS and measure their area results, but we have not reached a point on the AMC dialect side to get comparable area results. The major limiting factors here are the Calyx primitive issues described earlier and the lack of an optimization pass that maps to memory primitives.

As a result, the majority of our evaluation is more qualitative than quantitative. Every lowering pass and optimization is verified by the MLIR verifiers to be syntactically valid. We also checked all of our test cases to ensure each of the lowering passes makes sense.

The quality of our IR is evaluated by its expressiveness and ability to provide optimization opportunities. The wide range of memory types shown as examples above shows the expressiveness of our interface. One could also imagine any arbitrary combination of these concepts, such as a memory that has both latency-sensitive and latency-insensitive ports accessing the same banks. The memory aggregation optimization shows one such simple optimization which could potentially significantly improve area results for UltraRAMs that have a fixed 72b data width.

### Future Work

One of the biggest limitations we faced when lowering AMC dialect to Calyx is the limited support of certain memory operations in Calyx itself. In particular, Calyx lacks support for multi-ported memories, memories with a specific access latency, and memories that will map to a specific FPGA primitive. These should be able to be added using an external Calyx library, but this is not currently supported by by the Calyx MLIR dialect which we are lowering to.

As the future work we are planning to extend Calyx to support these operations so we can implement more functionality in AMC dialect and showcase more benefits of it over existing HLS tools. The lowering pass will also need to be more robust to support a wide range of optimizations and the memory interface will need to be interfaced with a scheduler of some sort to actual generate full designs. Comprehensive evaluation of AMC dialect and implementation of more optimizations is another major part of the future work.
