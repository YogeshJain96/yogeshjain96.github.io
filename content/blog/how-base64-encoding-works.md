---
title: "How Base64 Encoding Works?"
date: 2024-03-15T23:09:45+05:30
description: "Let's understand the Base64 Encoding under the hood."
tags: ["how-it-works", "base64", "encoding"]
cover:
  image: images/how-base64-encoding-works/cover.png
---

In my everyday work with Kubernetes, I often encounter Kubernetes secrets containing strings of seemingly random characters ending with equal signs. These are Base64 encoded texts. This made me curious: What exactly is Base64 encoding, and how does it work its magic?

So, one day, I decided to learn how Base64 actually works. It turns out to be a pretty simple process where the data (normal text and numbers) are converted into a special format that machines can easily read and use. Even though it's not super secure like encryption, it's really handy for keeping things organized and safe in places like Kubernetes.

### What exactly is Base64 Encoding?

Base64 is a method used to change any type of data, like text or pictures, into a format that's safe and easy for machines to handle. It's called "Base64" because it uses 64 different characters to represent data. These characters include letters (both uppercase and lowercase), numbers, and symbols like `+` and `/`. Base64 encoding ensures that the data is not altered during transmission and helps maintain data integrity as it passes through various systems and networks.

### How does Base64 Encoding Work?

Let's quickly break down into how Base64 encoding works step by step:

#### 1. Convert to ASCII
Convert each character in the string to its ASCII value.

#### 2. Convert ASCII to Binary
Convert each ASCII value to its 8-bit binary equivalent.

#### 3. Combine Binary Values
Combine all the 8-bit binary values into a single continuous binary string.

#### 4. Split into 6-bit Segments
Split the continuous binary string into segments of 6 bits each.

#### 5. Add Padding (if necessary)
If the final segment is less than 6 bits, add padding bits `0` to make it a full 6-bit segment.

#### 6. Convert 6-bit Segments to Decimal
Convert each 6-bit binary segment to its decimal equivalent.

#### 7. Map Decimal to Base64 Characters
Map each decimal value to its corresponding Base64 character using the Base64 index table, which consists of uppercase letters, lowercase letters, digits, plus `+`, & slash `/`

#### 8. Combine Base64 Characters
Combine all the Base64 characters obtained from the previous step into a single string.

#### 9. Add Padding Characters
If necessary, add `=` characters to the end of the Base64 string to make the total length a multiple of 4 characters.

### Example: Steps to Encode "Hello" into Base64

Here's a breakdown of how the word "Hello" is encoded into Base64 string:

<style>
  .custom-table td {
  min-width:0 !important;
  padding:0 !important;
  border: 1 !important;
  border-bottom: 0 !important;
  text-align: center;
}
.custom-table th{
  border-bottom: 0 !important;
}
</style>

<table border="1" class="custom-table">
  <thead>
    <tr style="font-weight:bold;">
      <th rowspan="2" scope="col">Source ASCII text</th>
      <th scope="col">Character</th>
      <td colspan="8">H</td>
      <td colspan="8">e</td>
      <td colspan="8">l</td>
      <td colspan="8">l</td>
      <td colspan="8">o</td>
    </tr>
    <tr>
      <th scope="col">ASCII Code</th>
      <td colspan="8">72</td>
      <td colspan="8">101</td>
      <td colspan="8">108</td>
      <td colspan="8">108</td>
      <td colspan="8">111</td>
    </tr>
    <tr>
      <th colspan="2" scope="col">Binary Equivalent(8 bits)</th>
      <td>0</td><td>1</td><td>0</td><td>0</td><td>1</td><td>0</td><td>0</td><td>0</td>
      <td>0</td><td>1</td><td>1</td><td>0</td><td>0</td><td>1</td><td>0</td><td>1</td>
      <td>0</td><td>1</td><td>1</td><td>0</td><td>1</td><td>1</td><td>0</td><td>0</td>
      <td>0</td><td>1</td><td>1</td><td>0</td><td>1</td><td>1</td><td>0</td><td>0</td>
      <td>0</td><td>1</td><td>1</td><td>0</td><td>1</td><td>1</td><td>1</td><td>1</td>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="3" scope="row">Base64 encoded text</th>
      <th scope="row">Grouped 6-bit Units</th>
      <td colspan="6">010010</td>
      <td colspan="6">000110</td>
      <td colspan="6">010101</td>
      <td colspan="6">101100</td>
      <td colspan="6">011011</td>
      <td colspan="6">000110</td>
      <td colspan="8">1111<mark>00</mark></td>
    </tr>
    <tr>
      <th scope="row">Decimal</th>
      <td colspan="6">18</td><td colspan="6">6</td><td colspan="6">21</td><td colspan="6">44</td><td colspan="6">27</td><td colspan="6">6</td><td colspan="6">60</td>
    </tr>
    <tr style="font-weight:bold;">
      <th scope="row">Character</th>
      <td colspan="6">S</td><td colspan="6">G</td><td colspan="6">V</td><td colspan="6">s</td><td colspan="6">b</td><td colspan="6">G</td><td colspan="6">8</td>
    </tr>
  </tbody>
</table>

**1. Convert "Hello" to ASCII**
First, convert each character in the string "Hello" to its ASCII value:

- `H` -> `72`
- `e` -> `101`
- `l` -> `108`
- `l` -> `108`
- `o` -> `111`

**2. Convert ASCII to Binary**
Next, convert each ASCII value to its 8-bit binary equivalent:

- `72`  -> `01001000`
- `101` -> `01100101`
- `108` -> `01101100`
- `108` -> `01101100`
- `111` -> `01101111`

**3. Combine Binary Values**
Combine these binary values into a single binary string:

`0100100001100101011011000110110001101111`

**4. Split into 6-bit Segments**
Split the combined binary string into 6-bit segments:

`010010` `000110` `010101` `101100` `011011` `000110` `1111`

**5. Add Padding**
Since the last segment is less than 6 bits, add padding to make it a full 6-bit segment. Each padding bit is 0:

`010010` `000110` `010101` `101100` `011011` `000110` `1111[00]`

**6. Convert 6-bit Segments to Decimal**
Convert each 6-bit segment to its decimal equivalent:

- `010010` -> `18`
- `000110` -> `6`
- `010101` -> `21`
- `101100` -> `44`
- `011011` -> `27`
- `000110` -> `6`
- `111100` -> `60`

**7. Map Decimal to Base64 Characters**
Map each decimal value to the corresponding Base64 character using the Base64 index table:

> **Reference:** Base64 index table

| Index   | Character | Index    | Character | Index    | Character | Index    | Character |
| ------- | --------- | -------- | --------- | -------- | --------- | -------- | --------- |
| 0       | `A`       | 16       | `Q`       | 32       | `g`       | 48       | `w`       |
| 1       | `B`       | 17       | `R`       | 33       | `h`       | 49       | `x`       |
| 2       | `C`       | <mark>18 | <mark>S   | 34       | `i`       | 50       | `y`       |
| 3       | `D`       | 19       | `T`       | 35       | `j`       | 51       | `z`       |
| 4       | `E`       | 20       | `U`       | 36       | `k`       | 52       | `0`       |
| 5       | `F`       | <mark>21 | <mark>V   | 37       | `l`       | 53       | `1`       |
| <mark>6 | <mark>G   | 22       | `W`       | 38       | `m`       | 54       | `2`       |
| 7       | `H`       | 23       | `X`       | 39       | `n`       | 55       | `3`       |
| 8       | `I`       | 24       | `Y`       | 40       | `o`       | 56       | `4`       |
| 9       | `J`       | 25       | `Z`       | 41       | `p`       | 57       | `5`       |
| 10      | `K`       | 26       | `a`       | 42       | `q`       | 58       | `6`       |
| 11      | `L`       | <mark>27 | <mark>b   | 43       | `r`       | 59       | `7`       |
| 12      | `M`       | 28       | `c`       | <mark>44 | <mark>s   | <mark>60 | <mark>8   |
| 13      | `N`       | 29       | `d`       | 45       | `t`       | 61       | `9`       |
| 14      | `O`       | 30       | `e`       | 46       | `u`       | 62       | `+`       |
| 15      | `P`       | 31       | `f`       | 47       | `v`       | 63       | `/`       |

- `18` -> `S`
- `6`  -> `G`
- `21` -> `V`
- `44` -> `s`
- `27` -> `b`
- `6` -> `G`
- `60` -> `8`

**8. Combine Base64 Characters**
Combine the Base64 characters: `SGVsbG8`

**9. Add Padding Characters**
Base64 encoding requires the output to be a multiple of 4 characters. As required here, let's add `=` to the end to make it so.

So, the Base64 encoding of <mark>Hello</mark> is <mark>SGVsbG8=</mark>

### Base64URL

Base64URL is a variant of the Base64 standard tailored for use in filenames and URLs. Standard Base64 includes the characters `+`, `/`,& `=`, which can interfere with their reserved meanings in filesystems and URLs. Base64URL replaces `+` with `-`, `/` with `_`,& removes the trailing `=` padding when not necessary, ensuring the encoded data can be used in URLs without any issues.

### Conclusion

Base64 encoding is a simple yet powerful tool to ensure data integrity and compatibility across different systems and platforms. By converting binary data into a text-friendly format, it allows for smooth data transfer and storage without any risk of data corruption, making it very useful for applications like emails and web development. Additionally, Base64 encoding is easily reversible, allowing for straightforward decoding back to the original data.
