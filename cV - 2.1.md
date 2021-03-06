# Correlation Vector v2.1

**Correlation Vector** (a.k.a. **cV**) is a tracing standard that evolved internally at Microsoft, based on a light weight vector clock.

This document covers:

- the design goals of the cV
- the format used on the wire when transmitting the vector across components involved in a trace
- the protocols used for managing the vector and the specific operators involved

## Design Goals

The design of the vector is optimized towards addressing the scenarios and constraints discussed in this document ([Link](Scenarios.md)).

Here are the key design goals:

1. Track causality (partial order) of the flow
2. Provide a simple sort independent of system clock time
3. Provide the sort/causality tracking capabilities for any arbitrary subset of events in the trace
4. Minimize wire cost for upstream components (that tend to use metered connections)
5. Simple to understand and implement

The robust sort and causality tracking requirements are based on the need to support both traditional debugging scenarios as well as business intelligence processing that relies on event telemetry that is emitted as part of a trace.

## Format

Represented by the following simply grammar ABNF specification

1. vector = base delim suffix
2. base = 21char lastChar
3. char = %x30-39 / %x41-5A / %61-7A / %x2F / %2B ; base64 characters
4. lastChar = "A" / "Q" / "g" / "w" ; base64 characters with four least significant bits of zero
5. suffix = elem / elem bang / elem delim suffix
6. elem = 1*10(%x30-39) ; decimal 4 byte unsigned counter
7. delim = %x2E ; '.'
8. bang = %x21 ; '!'

Less formally, the cV has the form **_{Base}.{Vector}_** where **_{Base}_** is a 128-bit base64 encoded value (with output padding "==" removed) and **_{Vector}_** is a "."-_delimited_ string of elements where each element is a 4 byte unsigned integer encoded in decimal, and is optionally terminated by the '!' character.

**Maximum length**: 128 bytes (assuming a UTF-8 encoding)

**Note**: The following characters are reserved for the specification { ‘.’, ‘!’ }

**Note**: The encoded **_{Base}_** consists of 22 base64 characters and, if left unrestricted, technically allows for 132-bit values. In order to support interoperability with tracing standards that use UUIDs, the **_{Base}_** value is restricted to 128-bits assuring reliable conversion to/from the UUID format. This is the reason the last element of the **_{Base}_** is restricted to a subset of allowable base64 characters.

### Examples

1. PmvzQKgYek6Sdk/T5sWaqw.0
2. e8iECJiOvUGPvOVtchxG9g.1.23
3. CgOLQOn9Gkmd4pM720ciZA.1.15.3226329855.4111101367.10.23.8.3226332926.1671828776.2345.12.3.243.544.3226336576.3422508575.23.1.34!

## Operators

The Operators defined on the cV are designed to support tracing while fulfilling the above-mentioned design goals.

To help define the vector operators are the following definitions and notations:

- X is a valid base, i.e. 128-bit base64 encoded value with output padding "==" removed
- V is a valid vector
- N is a valid 4-byte unsigned integer encoded in decimal

### Seed

**Definition**: {} => X.0

Invoked when initializing the vector.

### Increment

**Definition**: V.N => V.(N+1)

The Increment operator may be invoked arbitrarily to denote a logical tick. But it must be invoked prior to the creation of a new outgoing control flow to generate a unique vector value suitable for transmission such as, on an outgoing HTTP/RPC call.

Failure to increment on an outgoing span may result in child spans with identical vector values which breaks trace reconstruction.

If an increment is not possible, the workaround is to have the component that receives the vector value implement a Spin operation instead of an Extend (see below).

### Increment Examples

1. PmvzQKgYek6Sdk/T5sWaqw.0 => PmvzQKgYek6Sdk/T5sWaqw.1
2. e8iECJiOvUGPvOVtchxG9g.1.23 => e8iECJiOvUGPvOVtchxG9g.1.24
3. CgOLQOn9Gkmd4pM720ciZA.1.15.3226329855.4111101367.10.23.8.3226332926.1671828776.2345.12.3.243.544.3226336576.3422508575.23.1.34 => CgOLQOn9Gkmd4pM720ciZA.1.15.3226329855.4111101367.10.23.8.3226332926.1671828776.2345.12.3.243.544.3226336576.3422508575.23.1.35

### Extend

The Extend operator is to be used once on an incoming control flow, to extend the vector to create a vector element for the receiving span, such as on an incoming HTTP/RPC call.

**Definition**: V => V.0

### Extend Examples

1. PmvzQKgYek6Sdk/T5sWaqw.1 => PmvzQKgYek6Sdk/T5sWaqw.1.0
2. e8iECJiOvUGPvOVtchxG9g.1.23 => e8iECJiOvUGPvOVtchxG9g.1.23.0
3. CgOLQOn9Gkmd4pM720ciZA.1.15.3226329855.4111101367.10.23.8.3226332926.1671828776.2345.12.3.243.544.3226336576.3422508575.23.1 => CgOLQOn9Gkmd4pM720ciZA.1.15.3226329855.4111101367.10.23.8.3226332926.1671828776.2345.12.3.243.544.3226336576.3422508575.23.1.0

### Spin

The Spin operator is used instead of the Extend operator, in situations where the parent span cannot atomically increment the vector for each child span. 

Here are some common examples:

1. Publisher-consumer control flow patterns where the parent span in the publisher may not aware of the number of consumers, which can be greater than 1
2. Components that consume queues denoting incoming control flows and which support multiple dequeue retry attempts and without the capability for maintaining local state for tracking previous dequeue attempts in the current span.
3. Bad actor components which repeatedly pass along the same vector value on outgoing control flows without increments.
4. Very high throughput components where the interlocked increment operation is a significant overhead.

In such instances, without a uniquely incremented value on each incoming flow, the uniqueness constraint that promulgates that each span has its own unique cV is violated. This impacts downstream diagnostics as identical vector values are repeated on multiple spans.

The Spin operator mitigates this, by inserting an element with entropy so that repeated invocations result in unique values. However purely using entropy erodes the utility of the vector when it is used to provide a sort during data processing. For this reason, the construction precedes this with an element derived from UTC time. This supports the sort property of the vector while reducing the chances of collision.

**Definition**: V => V.A.B.0

- A is an unsigned integer derived from UTC time as follows:
  - If x is the UTC time in Ticks, 
    - Drop the least significant N bits of x to create a tick that increments, where N is the amount of bits dropped based on the Spin Parameter **Spin Counter Interval**.
    - Select the least significant N bits from the output of the previous step that yields a N-bit coarse tick counter where N is the amount of bits indicated in the Spin Parameters in **Spin Counter Periodicity**.

    Using a coarse spin interval, A therefore reflects a coarse tick counter that increments approximately every 6.5 milliseconds and overflows once in a little over 328 days.
- B is a randomly generated unsigned integer whose size is indicated in the Spin Parameter **Spin Entropy**. 32 bits of entropy yields a probability of collision of 1.15% for 10,000 trials. 

#### Spin Parameters

The output of a cV can be changed to output different values.

**Spin Counter Interval**: There are two different options for this:
- **Coarse** indicates that the 24 least significant bits of the UTC time counter are dropped.
- **Fine** indicates that the 16 least significant bits of the UTC time counter are dropped.

If the computer increments UTC time based on ticks, where 10 million ticks is equal to 1 second, **Coarse** will increment the spin counter every 1.67 seconds \(2\^24\*10\^\-7 seconds\) and **Fine** will increment the spin counter every 6.5 milliseconds \(2\^16\*10\^\-7 seconds\).

**Spin Counter Periodicity**: This indicates how many bits of the counter value described in the defined value A will be stored.
- **None** stores no \(0\) bits of the counter.
- **Short** stores 16 bits (2 bytes).
- **Medium** stores 24 bits (3 bytes).
- **Long** stores 32 bits (4 bytes).

**Spin Entropy**: This indicates how many bits of entropy described in the defined value B should be generated.
- **None** generates no \(0\) bits of entropy.
- **One** generates 8 bits (1 byte) of entropy.
- **Two** generates 16 bits (2 bytes) of entropy.
- **Three** generates 24 bits (3 bytes) of entropy.
- **Four** generates 32 bits (4 bytes) of entropy.

#### Spin Examples

1. PmvzQKgYek6Sdk/T5sWaqw.1 => PmvzQKgYek6Sdk/T5sWaqw.1.3226329855.4111101367.0
2. e8iECJiOvUGPvOVtchxG9g.1.23 => e8iECJiOvUGPvOVtchxG9g.1.23.3226332926.1671828776.0

## Termination

All operator implementations will include a check if the output exceeds 127 bytes in length. If it does, all operators will return the input vector appended with a '!' character. 

The presence of the trailing '!' character indicates that a previous attempt to mutate the vector failed and the vector should therefore be considered immutable. So any operator that receives an input vector with a trailing '!' character simply returns the same value. 

**Note**: The recommended approach when encountering terminated vectors would be to seed a new correlation vector and associate the terminated vector with the new one using additional telemetry metadata.


