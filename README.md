# Flexible Polyline encoding

The flexible polyline encoding is a lossy compressed representation of a list of coordinate pairs or
coordinate triples.

It achieves that by:

1. Reducing the decimal digits of each value.
2. Encoding only the offset from the previous point.
3. Using variable length for each coordinate delta.
4. Using 64 URL-safe characters to display the result.

The encoding is a variant of [Encoded Polyline Algorithm Format]. The advantage of this encoding
over the original are the following:

* Output string is composed by only URL-safe characters, i.e. may be used without URL encoding as 
  query parameters.
* Floating point precision is configurable: This allows to represent coordinates with
  precision up to microns (5 decimal places allow meter precision only).
* It allows to encode a 3rd dimension with a given precision, which may be a level, altitude, 
  elevation or some other custom value.

## Specifications

An encoded flexible polyline is composed by two main parts: A header and the actual polyline data. 
The header always starts with a version number that refers to the specifications in use. A change in 
the version may affect the logic to encode and decode the rest of the header and data. v.1 is the 
only version currently defined and this is the version assumed in the rest of the document.

```[header version][header content][data]```

### Encoding

Both header and data make use of variable length integer encoding.

Every input integer is converted in one or more chunks of 6 bits where the highest bit is a control 
bit while the remaining five store actual data. Each of these chunks gets encoded separately as a 
printable character using the following character set:

```ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-_```

Where `A` represents 0  and `_` represents 63.

#### Encoding an unsigned integer

The variable encoding uses the highest bit (the sixth in this case) as control bit: When it's set to 
`1` it means another chunk needs to be read. Here is the algorithm to encode an unsigned integer:

1. Given the binary representation of the number, split it in chunks of 5 bits.
2. Convert every chunk from right to left.
 1. Every chunk that is followed by another chunk will be in OR with `0x20` (set the sixth bit to 
    `1`) and then encoded as a character.
 2. The last chunk (leftmost) is encoded as a character directly since the sixth bit is already set 
    to `0`.

#### Encoding a signed integer

In the binary representation of signed integers, the least significant bit stores the sign and the 
subsequent bits encode the number value resp. `abs(n)-1` for `n < 0`. That is, a positive integer 
`p` is stored as `2*p` while a negative integer `n` is stored as `2*abs(n)-1`. So for example, the 
number 7 is represented as `1110` and -7 as `1101`. After this transformation, the normal unsigned 
integer encoding algorithm can be used.


### Header version

The first unsigned varint of the encoded string refers to the specification version.

### Header content

It's encoded as an unsigned varint. The bits of the header content have the following structure:

```
bit   [10              7] [6          4] [3       0]
value [3rd dim precision] [3rd dim flag] [precision]
```

#### precision

Refers to the precision (decimal digits after the comma) of the latitude and longitude coordinates. 
It is encoded as an unsigned integer with a range 0 to 15.

#### 3rd dim flag

This flag specifies whether the third dimension is present and what meaning it has. It's encoded as 
an unsigned integer with a range 0 to 7.

Possible values are:

  * 0 – absent
  * 1 – level
  * 2 – altitude
  * 3 – elevation
  * 4 – *reserved1*
  * 5 – *reserved2*
  * 6 – custom1
  * 7 – custom2

Note that the concrete unit used for the 3rd dimension is not specified, and depends on how the format is used.
It is recommended to make sure it is properly documented (e.g. elevation could be represented in WGS84 or EGM96).


#### 3rd dim precision

Refers to the precision (decimal digits after the comma) of the third dimension. Possible values are 
0 to 15.

### Data

All data values need to be normalized before encoding by transforming them to integers with the 
given precision. For example if precision is 5, the value 12.3 becomes 1230000.

The data section is composed by a sequence of signed varints grouped in tuples of the same size (2 
if the 3rd dimension flag is absent and 3 otherwise).

The first tuple contains the first set of normalized coordinates, while any other subsequent tuple 
contains the offset between two consecutive values on the same dimension.

```Lat0 Lng0 3rd0 (Lat1-Lat0) (Lng1-Lng0) (3rdDim1-3rdDim0) ...```

## Example

The following coordinates

```
(50.10228, 8.69821), (50.10201, 8.69567), (50.10063, 8.69150), (50.09878, 8.68752)
```

are encoded with precision 5 as follows:

```
B F oz5xJ 67i1B 1B 7P zI ha xL 7Y
```

* The initial `B` is the header version (v.1)
* The second letter `F` is the header, corresponding to the precision 5 and no 3rd dimension set.
* The rest of the string are the encoded coordinates.

## Pseudocode

The following pseudocode illustrates the steps needed to decode an encoded string and
might also be a helpful template for an actual implementation. Note that error handling
is not taken care of in the code.

The first function that is needed for a decoder is retuning the index of a single
character in the character set. E.g. for the first character 'A' it should return 0.

This can be implemented in an efficient way by converting the character into its
ASCII-code and using this as an index in a decoding table.
E.g. for the character 'A' the ASCII code is 65, so the decoding table should be
initialized with a 0 at this index.

```commandline
function decode_integer
    input: a character from the encoded string
    output: the index of the character in the character set

    DECODING_TABLE := [
        -1, -1, -1, -1, -1, -1, -1, -1, -1, -1,
        -1, -1, -1, -1, -1, -1, -1, -1, -1, -1,
        -1, -1, -1, -1, -1, -1, -1, -1, -1, -1,
        -1, -1, -1, -1, -1, -1, -1, -1, -1, -1,
        -1, -1, -1, -1, -1, 62, -1, -1, 52, 53,
        54, 55, 56, 57, 58, 59, 60, 61, -1, -1,
        -1, -1, -1, -1, -1,  0,  1,  2,  3,  4,
         5,  6,  7,  8,  9, 10, 11, 12, 13, 14,
        15, 16, 17, 18, 19, 20, 21, 22, 23, 24,
        25, -1, -1, -1, -1, 63, -1, 26, 27, 28,
        29, 30, 31, 32, 33, 34, 35, 36, 37, 38,
        39, 40, 41, 42, 43, 44, 45, 46, 47, 48,
        49, 50, 51
    ]

    return DECODING_TABLE[asci_code(charater)]

end_function
```

Now the decoding of the first two header bytes can be implemented:

```commandline
function decode_header
    input: encoded_string
    output: a tuple of 3 values (precision_2d, type_3d, precision_3d)

    header_version = decode_integer(encoded_string[0])
    header_content = decode_integer(encoded_string[1])

    precision_2d = header_content & 0xF
    type_3d = (header_content >> 4) & 0x7
    precision_3d = (header_content >> 7) & 0xF

    return Tuple(precision_2d, type_3d, precision_3d)

end_function
```

After the two header bytes the encoded coordinates start. More precisely,
the sequence of signed coordinate deltas in integer space.

The function below decodes these bytes into a sequence of signed integer values:

```commandline
function decode_signed_deltas
    input: encoded_deltas (= encoded string without first two header bytes)
    output: sequence of signed integer values

    values := array()
    next_value := 0
    shift := 0
    for character in encoded_deltas
        chunk := decode_integer(character)
        is_last_chunk := (chunk & 0x20) == 0
        chunk_value := chunk & 0x1F

        # prepend the chunk value to next_value:
        next_value := (chunk_value << shift) | next_value
        shift := shift + 5

        if is_last_chunk
            # Convert chunk_value to a signed integer:
            if next_value & 1 == 1  # if first bit is 1, value is negative
                signed_value := - ((next_value + 1) >> 1)
            else
                signed_value := next_value >> 1
            end_if
            values.append(signed_value)
            next_value := 0
            shift := 0
        end_if

    end_for

    return values

end_function
```

Given the function above all that remains to be done is to group the returned sequence into
tuples of size 2 or 3 and convert them into floats given the precision defined in the header.

This is the pseudocode for the case of a 2d polyline:

```commandline
function decode_flexpolyline_2d
    input: encoded_deltas  # (= encoded string without first two header bytes)
           precision_2d       # integer with number of decimal digits
    output: array of (lat, lon) coordinate tuples

    deltas := decode_signed_deltas(encoded_coordinates)

    coordinates := array()
    lat := 0
    lon := 0
    for i in [0, 1, ... length(deltas) / 2]
        lat := lat + deltas[2 * i]
        lon := lon + deltas[2 * i + 1]
        coordinates.append(Tuple(lat / 10 ** precision_2d, lon / 10 ** precision_2d))
    end_for

    return coordinates

end_function
```

## My favorite language is not supported. What now?

Feel free to contribute an implementation. You can either use C-bindings, or
provide an implementation in your language. Take a look at the implementations in other languages
already available. Usually, the encoding/decoding is very straight-forward. The interface should
match roughly the following API:

```
encode(coordinates, precision, third_dimension, third_dimension_precision) -> string;
decode(string) -> coordinates;
get_third_dimension(string) -> third_dimension;
```

To test your implementation, use the polylines defined in [test/original.txt]. Depending on the 
round function available in the language the expected encoded and decoded files need to be used:

* Round to nearest, ties away from zero:
  * [test/round\_half_up/encoded.txt]
  * [test/round\_half_up/decoded.txt]
* Round to nearest, ties to even:
  * [test/round\_half_even/encoded.txt]
  * [test/round\_half_even/decoded.txt]

Check that encoded the original data results in the encoded form, and that decoding it again results 
in the decoded form.
Format of the unencoded data is:

* 2d: `{(precision2d); [(lat, lon), ..., (lat, lon), ]}`
* 3d: `{(precision2d, precision3d, type3d); [(lat, lon, z), ..., (lat, lon, z), ]}`

Floating point numbers are printed with 15 digits decimal precision. Be aware that encoding is 
lossy: Decoding an encoded polyline will not always yield the original, and neither will encoding a 
decoded polyline result in the same encoded representation.

### Implementation hints:

* 32-bit floats and integers are not sufficient for encodings with high precision,
use 64-bits instead

## TODO

* Extend provided tests. We should cover all combinations of parameters.
* Add C-bindings for people who just want to wrap them.

[Encoded Polyline Algorithm Format]: https://developers.google.com/maps/documentation/utilities/polylinealgorithm
[test/original.txt]: test/original.txt
[test/round\_half_up/encoded.txt]: test/round_half_up/encoded.txt
[test/round\_half_up/decoded.txt]: test/round_half_up/decoded.txt
[test/round\_half_even/encoded.txt]: test/round_half_even/encoded.txt
[test/round\_half_even/decoded.txt]: test/round_half_even/decoded.txt

## License

Copyright (C) 2019 HERE Europe B.V.

See the [LICENSE](./LICENSE) file in the root of this project for license details.
