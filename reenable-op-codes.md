# Restore disabled script op codes

Version 0.1, 2018-01-19 - DRAFT FOR DISCUSSION

## Draft discussion notes

For the purposes of discussion of this draft additional notes are contained throughout with the heading *DRAFT DISCCUSION*.  These are
intended to be removed from the finalalized version of this document.

Optional rules are denoted by *RULE OPTION* where it is intended one the presented options will be adopted following consensus within
the workgroup.


## Introduction

This document describes proposed requirements for reactivating several script op codes.  In 2011 the discovery of
two serious bugs in `OP_LSHIFT` and `OP_RETURN` prompted the deactivation of these and 13 additional op codes.
The following list of disabled op codes is broken down by category.

##### Splice operations

|Word       |OpCode |Hex |Input         |Output  | Description                                                      |
|-----------|-------|----|--------------|--------|------------------------------------------------------------------|
|OP_CAT     |126    |0x7e|x1 x2         |out     |Concatenates two byte strings                                     |
|OP_SUBSTR  |127    |0x7f|in begin size |out     |Returns a section of a string                                     |
|OP_LEFT    |128    |0x80|in size       |out     |Keeps only characters left of the specified point in the string   |
|OP_RIGHT   |129    |0x81|in size       |out     |Keeps only characters right of the specified point in the string  |


##### Bitwise logic

|Word       |OpCode |Hex |Input         |Output  | Description                                                      |
|-----------|-------|----|--------------|--------|------------------------------------------------------------------|
|OP_INVERT  |131    |0x83|in            |out     |Flips all bits of the input                                       |
|OP_AND     |132    |0x84|x1 x2         |out     |Boolean *AND* beteween each bit of the inputs                     |
|OP_OR      |133    |0x85|x1 x2         |out     |Boolean *OR* beteween each bit of the inputs                      |
|OP_XOR     |134    |0x86|x1 x2         |out     |Boolean *EXCLUSIVE OR* beteween each bit of the inputs            |


##### Arithmetic

|Word       |OpCode |Hex |Input         |Output  | Description                                                      |
|-----------|-------|----|--------------|--------|------------------------------------------------------------------|
|OP_2MUL    |141    |0x8d|in            |out     |The input is multiplied by 2                                      |
|OP_2DIV    |142    |0x8e|in            |out     |The input is divided by 2                                         |
|OP_MUL     |149    |0x95|a b           |out     |*a* is multiplied by *b*                                          |
|OP_DIV     |150    |0x96|a b           |out     |*a* is divided by *b*                                             |
|OP_MOD     |151    |0x97|a b           |out     |return the remainder after *a* is divided by *b*                  |
|OP_LSHIFT  |152    |0x98|a b           |out     |shifts *a* left by *b* bits, preserving sign                      |
|OP_RSHIFT  |152    |0x99|a b           |out     |shifts *a* right by *b* bits, preserving sign                     |


### Proposed op codes to reenable

It is proposed to reintroduce these op codes (or equivalent functionality) in a staged process.  The first stage being
to enable a limited subset in the May hard fork. 

Splice operations: `OP_CAT`, `OP_SPLIT`**

Bitwise logic: `OP_AND`, `OP_OR`, `OP_XOR`

Arithmetic: `OP_DIV`, `OP_MOD`

New: 
* `x OP_BIN2NUM -> n` - convert a binary array `x` into a valid (canonical) numeric element
* `n m OP_NUM2BIN -> out` - convert a numeric value `n` into a byte array of length `m`

* `data sig addr OP_DATASIGVERIFY -> data` - Check that signature verification of `data` with `sig` is valid and produces the pubkey whose hash160 is `addr` 

** A new operation, `OP_SPLIT`, is proposed as a replacement for `OP_SUBSTR`, `OP_LEFT`and `OP_RIGHT`. All three operations can be
simulated with varying combinations of `OP_SPLIT`, `OP_SWAP` and `OP_DROP`.

*** futher discussion of the purpose of these new operation under bitwise operations.

## <a name="data-types"></a>Script data types

It should be noted that in script operation data value on the stack are interpreted as either binary strings (i.e. an array of bytes) 
or numbers.  **All data on the stack is interpreted as an array of bytes unless specifically stated as being interpreted as numeric.**

Numeric type have specific limitations:
1. The used encoding is little endian with an explicit sign bit (the highest bit of the last byte).
2. They cannot exceed 4 bytes in length.
3. They must be encoded using the shortest possible byte length (no zero padding)
    3. There is one exception to rule 3: if there is more than one byte and the most significant bit of the 
        second-most-significant-byte is set it would conflict with the sign bit. In this case a single 0x00 or 0x80 byte is allowed
        to the left.
4. Negative zero is not allowed.
    
The newly proposed opcode `x OP_BIN2NUM -> out` can be used convert a binary array into a canonical number where required.

The newly proposed opcode `x n OP_NUM2BIN` can be used to convert a number into a zero padded binary array of length `n` 
whilst preserving the sign bit.

**Endian notation**

For human readability where hex strings are presented in this document big endian notation is used.  That is: 0x0100 represents decimal 256
not decimal 1


## Risks and philosophical approach

In general the approach taken is a minimalist one in order limit edge cases as much as possible.  Where it is possible
for a primitive op code used in conjuction with existing op codes to be combined to produce several more complex operations that is
preferred over a set of more complex op codes.  Input conditions that create ambiguous or undefined behaviour should fail fast.  

Each op code should be examined for the following risk conditions and mitigating behaviour defined explcitly:
* Operand byte length mismatch.  Where it would be normally expected that two operands would be of matching byte lengths
the resultant behaviour should be defined.
* Signed integer.  Whether signed integers are permitted operands and whether any special handling is required.
* Stack size impact.  Both number of elements and total size of elements. 
* Overflows.  Defined behaviour in the instance that result of the operation exceeds MAX_SCRIPT_ELEMENT_SIZE
* Empty byte vector operands.  Whether empty byte vectors should be allowed as a representation of zero.
* Empty byte vector output.  Note that an operation that outputs an empty byte array has effectively pushed `false` onto the stack.
  If this is the last operation in a script or if a conditional operator immediately follows the script author must consider this possibility.
  This is currently the case for many existing op codes however so it is consistent to continue with allowing this behaviour.

## Definitions

* *Stack memory use* - sum of the size of the elements on the stack - gives an indication of impact on memory use
* *Operand order* - in keeping with convention where multiple operands are specified the top most stack item is the 
last operand.  e.g. `x1 x2 OP_CAT` --> `x2` is the top stack item and `x1` is the next from the top
* *empty byte array* - throughout this document `OP_0` is used as a convenient representation of an empty byte array.  Whilst it is
 a push data op code it's effect is to push an empty byte array to the stack.

## Specification

Global failure conditions apply to all operations. These failure conditions must be checked by the implementation when 
it is possible that they will occur:
* for all e : elements on the stack, `0 <= len(e) <= MAX_SCRIPT_ELEMENT_SIZE`
* for each operator, the required number of operands are present on the stack when the operand is executed

These unit tests should be included for every operation:
1. executing the operation with an input element of length greater than `MAX_SCRIPT_ELEMENT_SIZE` will fail
2. executing the operation with an insufficient number of operands on the stack causes a failure


Operand consumption:

In all cases where not explicitly stated otherwise the operand stack elements are consumed by the operation and replaced with the output.

## Splice operations

### OP_CAT

    Opcode (decimal): 126
    Opcode (hex): 0x7e

Concatenates two operands.

    x1 x2 OP_CAT → out
        
Examples:
* `Ox11 0x2233 OP_CAT -> 0x112233`
    
The operator must fail if:
* `0 <= len(out) <= MAX_SCRIPT_ELEMENT_SIZE` - the operation cannot output elements that violate the constraint on the element size

Note that the concatentation of a zero length operand is valid

Impact of successful execution:
* stack memory use is constant
* number of elements on stack is reduced by one

The limit on the length of the output prevents the memory exhaustion attack and results in the operation having less 
impact on stack size than existing OP_DUP operators.

Unit tests:
1. `maxlen_x y OP_CAT → failure` – concatenating any operand except an empty vector, including a single byte value (e.g. `OP_1`), onto a maximum sized array causes failure
3. `large_x large_y OP_CAT → failure` – concatenating two operands, where the total length is greater than `MAX_SCRIPT_ELEMENT_SIZE`, causes failure
4. `OP_0 OP_0 OP_CAT → OP_0` – concatenating two empty arrays results in an empty array
5. `x OP_0 OP_CAT → x` – concatenating an empty array onto any operand results in the operand, including when `len(x) = MAX_SCRIPT_ELEMENT_SIZE`
6. `OP_0 x OP_CAT → x` – concatenating any operand onto an empty array results in the operand, including when `len(x) = MAX_SCRIPT_ELEMENT_SIZE`
7. `x y OP_CAT → concat(x,y)` – concatenating two operands generates the correct result

### OP_SPLIT

*`OP_SPLIT` replaces `OP_SUBSTR` and uses it's opcode.*

    Opcode (decimal): 127
    Opcode (hex): 0x7f


Split the operand at the given position.  This operation is the exact inverse of OP_CAT

    x n OP_SPLIT -> x1 x2

    where n is interpreted as a number

Examples:
* `0x001122 0 OP_SPLIT -> OP_0 0x001122`
* `0x001122 1 OP_SPLIT -> 0x00 0x1122`
* `0x001122 2 OP_SPLIT -> 0x0011 0x22`
* `0x001122 3 OP_SPLIT -> 0x00112233 OP_0`

Notes:
* this operator has been introduced as a replacement for the previous `OP_SUBSTR`, `OP_LEFT`and `OP_RIGHT`. All three operators can be
simulated with varying combinations of `OP_SPLIT`, `OP_SWAP` and `OP_DROP`.  This is in keeping with the minimalist philosophy where a single
primitive can be used to simulate multiple more complex operations.
* `x` is split at position `n`, where `n` is the number of bytes from the beginning
* `x1` will be the first `n` bytes of `x` and `x2` will be the remaining bytes 
* if `n == 0`, then `x1` is the empty array and `x2 == x`
* if `n == len(x)` then `x1 == x` and `x2` is the empty array.
* if `n > len(x)`, then the operator must fail.
* `x n OP_SPLIT OP_CAT -> x` - for all `x` and for all `0 <= n <= len(x)`
    
The operator must fail if:
* `!isnum(n)` - `n` is not a number
* `n < 0` - `n` is negative
* `n > len(x)` - `n` is too high

Impact of successful execution:
* stack memory use is constant (slight reduction by `len(n)`)
* number of elements on stack is constant

Unit tests:
* `OP_0 0 OP_SPLIT -> OP_0 OP_0` - execution of OP_SPLIT on empty array results in two empty arrays.
* `x 0 OP_SPLIT -> OP_0 x`
* `x len(x) OP_SPLIT -> x OP_0`
* `x (len(x) + 1) OP_SPLIT -> FAIL`

## Bitwise logic

The bitwise logic operators expect 'array of bytes' operands. The operands must be the same length. 
* In the case of 'array of bytes' operands `OP_CAT` can be used to pad a shorter byte array to an appropriate length.
* In the case of 'array of bytes' operands there the length of operands is not known until runtime an array of 0x00 bytes 
(for use with `OP_CAT`) can be produced using `OP_0 n OP_NUM2BIN`
* In the case of numeric operands `x n OP_NUM2BIN` can be used to pad a number to length `n` whilst preserving the sign bit.

### OP_AND

    Opcode (decimal): 132
    Opcode (hex): 0x84

Boolean *and* between each bit in the operands.

	x1 x2 OP_AND → out

Notes:
* where `len(x1) == 0 == len(x2)` the output will be an empty array.

The operator must fail if:
1. `len(x1) != len(x2)` - the length, in bytes, of the two values is not equal

Impact of successful execution:
* stack memory use reduced by `len(x1)`
* number of elements on stack is reduced by one

Unit tests:

1. `x1 x2 OP_AND -> failure`, where `len(x1) != len(x2)` - operation fails when length of operands not equal
2. `x1 x2 OP_AND -> x1 & x2` - check valid results

### OP_OR

    Opcode (decimal): 133
    Opcode (hex): 0x85

Boolean *or* between each bit in the operands.

	x1 x2 OP_OR → out
	
The operator must fail if:
1. `len(x1) != len(x2)` - the length, in bytes, of the two values is not equal

Impact of successful execution:
* stack memory use reduced by `len(x1)`
* number of elements on stack is reduced by one

Unit tests:
1. `x1 x2 OP_OR -> failure`, where `len(x1) != len(x2)` - operation fails when length of operands not equal
2. `x1 x2 OP_OR -> x1 | x2` - check valid results

### OP_XOR

    Opcode (decimal): 134
    Opcode (hex): 0x86

Boolean *xor* between each bit in the operands.

	x1 x2 OP_XOR → out
	
The operator must fail if:
1. `len(x1) != len(x2)` - the length, in bytes, of the two operands is not equal

Impact of successful execution:
* stack memory use reduced by `len(x1)`
* number of elements on stack is reduced by one

Unit tests:
1. `x1 x2 OP_XOR -> failure`, where `len(x1) != len(x2)` - operation fails when length of operands not equal
2. `x1 x2 OP_XOR -> x1 xor x2` - check valid results
    
## Arithmetic

#### Note about canonical form and floor division

Operands for all arithmetic operations are assumed to be numbers and must be in canonical form.  See [data types](#data-types) for more
information.

**Floor division**

Note: that when considering integer division and modulo operations with negative operands the rules applied in the C language and most
languages (with Python being a notable exception) differ from the string mathematical definition.  Script follows the C language set of
rules.  Namely:
1. Non-integer quotients are rounded towards zero
2. The equation `(a/b)*b + a%b == a` is satisfied by the results
3. From the above equation it follows that: `a%b == a - (a/b)*b`
4. In practice if `a` is negative for the modulo operator the result will be negative or zero or negative zero.



### OP_DIV

    Opcode (decimal): 150
    Opcode (hex): 0x96
    
Return the integer quotient of `a` and `b`.  If the result would be a non-integer it rounded *towards* zero.

    a b OP_DIV -> out
    
    where a and b are interpreted as numbers
    
The operator must fail if:
1. `!isnum(a) || !isnum(b)` - either operand is not a valid number
1. `b == 0` - `b` is equal to any type of zero

Impact of successful execution:
* stack memory use reduced (one element removed)
* number of elements on stack is reduced by one

Unit tests:
1. `a b OP_DIV -> failure` where `!isnum(a)` or `!isnum(b)` - both operands must be valid numbers
2. `a 0 OP_DIV -> failure` - division by positive zero (all sizes), negative zero (all sizes), `OP_0` 
3. `a b OP_DIV -> out` where `a < 0` the result must be negative or any form of zero. 
4. check valid results for operands of different lengths `1..4`
    
### OP_MOD

    Opcode (decimal): 151
    Opcode (hex): 0x97

Returns the remainder after dividing a by b.  The output will be represented using the least number of bytes required. 

	a b OP_MOD → out
	
	where a and b are interpreted as numbers
	
The operator must fail if:
1. `!isnum(a) || !isnum(b)` - either operand is not a valid number
1. `b == 0` - `b` is a negative number or equal to any type of zero

Impact of successful execution:
* stack memory use reduced (one element removed)
* number of elements on stack is reduced by one

Unit tests:
1. `a b OP_MOD -> failure` where `!isnum(a)` or `!isnum(b)` - both operands must be valid numbers
2. `a 0 OP_MOD -> failure` - division by positive zero (all sizes), negative zero (all sizes), `OP_0` 
3. `a b OP_MOD -> failure` where `a < 0`, `b < 0` - both operands must be positive
4. check valid results for operands of different lengths `1..4`

## New operations

#### DRAFT DISCUSSION

In order to facilitate the "operands must be equal length" rule for bitwise logic.  An additional operator is required to give script
authors a reasonable way of padding operands when required.  A clear distinction must be made between padding an array of bytes and padding
a number.
        
#### END DRAFT DISCUSSION

### OP_BIN2NUM

*`OP_BIN2NUM` replaces `OP_LEFT` and uses it's opcode*

    Opcode (decimal): 128
    Opcode (hex): 0x80

Convert the binary array into a valid numeric value, including minimal encoding.

    `x1 OP_BIN2NUM -> n`

See also `OP_NUM2BIN`.
    
Examples:
* `0x0000000002 OP_BIN2NUM -> 0x02`
* `0x800005 OP_BIN2NUM -> 0x85`

The operator must fail if:
1. the numeric value is out of the range of acceptable numeric values (currently size is limited to 4 bytes)

     
### OP_NUM2BIN

*`OP_NUM2BIN` replaces `OP_RIGHT` and uses it's opcode*

    Opcode (decimal): 129
    Opcode (hex): 0x81

Convert the numeric value into a binary array of a certain size, taking account of the sign bit.

    `n m OP_NUM2BIN -> x`
    
    where m and n are interpreted as numbers

See also `OP_BIN2NUM`.

Examples:
* `0x02 4 OP_NUM2BIN -> 0x00000002`
* `0x85 4 OP_NUM2BIN -> 0x80000005`

The operator must fail if:
1. `n` or `m` are not valid numeric values
2. `m < len(n)`. `n` is a valid numeric value, therefore it is already in minimal representation 
3. `m > MAX_SCRIPT_ELEMENT_SIZE` - the result would be too large

### OP_DATASIGVERIFY

*OP_DATASIGVERIFY uses a new opcode number*

    Opcode (decimal): 187
    Opcode (hex): 0xbb

OP_DATASIGVERIFY allows signed data to be imported into a script.  This data can then have many uses, depending on the rest of the script, such as deciding spendability of several possible addresses.  This opcode therefore enables the powerful blockchain concept of an "oracle" -- an entity that publishes authoritative statements about extra-blockchain events -- to be used in the Bitcoin Cash blockchain.  For an example use of how this opcode can be used to enable binary contracts on any security or betting on any quantitatively decidable event (such as a sports match) please [click here](https://medium.com/@g.andrew.stone/bitcoin-scripting-applications-decision-based-spending-8e7b93d7bdb9).  But this is just one example; as the Bitcoin Cash Script language grows in expressiveness, it is anticipated that this opcode will be used in many other applications.

Examples:

* Simplest DATASIGVERIFY:
  * output script:  `address OP_DATASIGVERIFY`
  * spend script: `data sig`

* Realistic example: Verify that `data` is signed by `daddr` (signature `daddrsig`) and equals `x`.  Also do normal p2pkh transaction verification on `addr` (with signature `addrSig` and public key `addrPub`):
  * output script: `daddr OP_DATASIGVERIFY x OP_EQUALVERIFY OP_DUP OP_HASH160 addr OP_EQUALVERIFY OP_CHECKSIG
  * spend script: `addrSig addrPub data daddrsig`

Implementation details:
When OP_DATASIGVERIFY is run, the stack should look like:

*top of stack*
* address
* signature
* data

If there are less then 3 items on the stack, the script fails.  If the address field is not 20 bytes, the script fails.

OP_DATASIGVERIFY first computes the double-sha256 hash of the byte string "Bitcoin Signed Message:\n" prepended to the supplied data (stack top - 2). This is the same operation as the Bitcoin Cash message signing RPC, and is the same algorithm as is used in OP_CHECKSIGVERIFY.  It then computes a pubkey from this hash and the provided signature (stack top - 1).  It then compares the hash160 (same as OP_HASH160) of this signature to the provided address (stack top).  If the comparison fails, the script fails.

Otherwise, the top 2 items are popped off the stack (leaving "data" on the top of the stack), and opcode succeeds.

#### OP_DATASIGVERIFY Reference Implementation

Please refer to [this github branch](https://github.com/gandrewstone/BitcoinUnlimited/tree/op_datasigverify) for a complete implementation.  But the opcode implementation is short enough to include here:
```c++
// This code sits inside the interpreter's opcode processing case statement
case OP_DATASIGVERIFY:
{
    if (stack.size() < 3)
        return set_error(serror, SCRIPT_ERR_INVALID_STACK_OPERATION);

    valtype &data = stacktop(-3);
    valtype &vchSig = stacktop(-2);
    valtype &vchAddr = stacktop(-1);

    if (vchAddr.size() != 20)
        return set_error(serror, SCRIPT_ERR_INVALID_STACK_OPERATION);

    CHashWriter ss(SER_GETHASH, 0);
    ss << strMessageMagic << data;

    CPubKey pubkey;
    if (!pubkey.RecoverCompact(ss.GetHash(), vchSig))
        return set_error(serror, SCRIPT_ERR_VERIFY);
    CKeyID id = pubkey.GetID();
    if (id != uint160(vchAddr))
        return set_error(serror, SCRIPT_ERR_VERIFY);
    popstack(stack);
    popstack(stack);
}
break;

```


## Reference implementation

TODO

## References

<a name="op_codes">[1]</a> https://en.bitcoin.it/wiki/Script#Opcodes
