---
title: >
  Additional
  Application Extensions for the CBOR Extended Diagnostic Notation (EDN)
abbrev: EDN Application Extensions
docname: draft-bormann-cbor-edn-app-ext-latest
# date: 2026-04-10

v: 3

keyword: Internet-Draft
cat: std
stream: IETF
consensus: true

venue:
  mail: "cbor@ietf.org"
  github: "cabo/edn-app-ext"
  latest: "https://cabo.github.io/edn-app-ext/draft-bormann-cbor-edn-app-ext.html"

author:
  -
    name: Carsten Bormann
    org: Universität Bremen TZI
    street: Postfach 330440
    city: Bremen
    code: D-28359
    country: Germany
    phone: +49-421-218-63921
    email: cabo@tzi.org

normative:
  I-D.ietf-cbor-edn-literals: edn
  IANA.cbor-diagnostic-notation: iana-edn
  STD68: abnf # RFC5234
  STD94: cbor # RFC8949
  RFC7405: abnf2
  RFC4648: base
  I-D.josefsson-rfc4648bis: basebis
  RFC9285: base45
  RFC9741: more
  IEEE754:
    target: https://ieeexplore.ieee.org/document/8766229
    title: IEEE Standard for Floating-Point Arithmetic
    author:
    - org: IEEE
    date: false
    seriesinfo:
      IEEE Std: 754-2019
      DOI: 10.1109/IEEESTD.2019.8766229
informative:

--- abstract

The CBOR Extended Diagnostic Notation (EDN), to be standardized in
draft-ietf-cbor-edn-literals,
provides "application extensions" as its main language extension point.

A number of application extensions are already defined in
draft-ietf-cbor-edn-literals itself and in draft-ietf-cbor-edn-e-ref.
The present document defines a number of additional application
extensions that have been batched up as a next step after completing
these specifications.
([^chore] Briefly List extensions.)

[^status]

[^status]: This -00 of an individual submission shows the approximate
    shape the first "batch" of application extensions could have, plus
    a number of registrations that could go into this batch.
    The latter provides a basis for a technical discussion of those
    registrations.

--- middle

Introduction        {#intro}
============

(See abstract.)

| Name             | Purpose                                                                                            |
| same             | Multiple literals for the same item, to be checked against each other                              |
| bytes            | Reinterpret text string as byte string                                                             |
| float            | Provide IEEE754-oriented literals for more floating point values                                   |
| tbd b32/h32/c32? | Create byte string from base32 representation, possibly beyond the two variants defined in {{-base}} |
| tbd b45          | Create byte string from base45 representation {{-base45}} |
{: #tbl-new title="Additional EDN application extensions defined in this document"}

[^discuss] We should also add application extensions for text
generation from bytes, such as b64c and b64u, along the lines of {{-more}}.

Conventions and Terminology
-----------

This specification uses terminology from {{-edn}}.
In particular, with respect to control operators, "target" refers to
the left-hand side operand, and "controller" to the right-hand side operand.
"Tool" refers to tools that produce, consume, or otherwise process EDN.
Note also that the data model underlying CBOR provides for text
strings as well as byte strings as two separate types, which are
then collectively referred to as "strings".

The term ABNF in this specification stands for the combination of
{{STD68}} and {{RFC7405}}, i.e., ABNF defined in this document allows use
of the case-sensitive extensions defined in {{RFC7405}}.

{::boilerplate bcp14-tagged-bcp14}

Application Extensions
=================

A table like Table 6 in {{-edn}}:
(Add text.)

| app-prefix | content of single-quoted string                  | result type                         |
|------------+--------------------------------------------------+-------------------------------------|
| same       | sequence of one or more data items               | data item                           |
| SAME       | (not used)                                       |                                     |
| bytes      | byte or text string(s)                           | byte string with concatenated bytes |
| BYTES      | (not used)                                       |                                     |
| float      | byte string (usually hex-encoded as text string) | floating point value                |
| FLOAT      | (not used)                                       |                                     |
{: #tab-prefixes title="App-prefix Values Defined in this Document"}

## same: multiple literals to be checked against each other

The "`same`" application extension receives a sequence of one or more
data items and throws an error if these data items are not the same.

Example:

~~~
$ edn-abnf -afloat,same -e "same<< float'47110815', 37128.08203125, 0x1.22102ap+15 >>"
37128.08203125
$ edn-abnf -afloat,same -e "same<< float'47110815', 37128.08203126 >>"
** cbor-diagnostic: same<<>>: 37128.08203125 not same as 37128.08203126: Argument Error
$ edn-abnf -asame -e "same<< h'a10101', <<{/alg/ 1: 1 /AES-GCM 128 /}>> >>"
h'A10101'
$ edn-abnf -asame -e "same<<1>>" # trivially true
1
~~~
{: post="fold"}

## bytes: Reinterpret text string as byte string

The "`bytes`" application extension receives a sequence of zero or more
strings (throwing an error if any of these data items are not text or
byte strings), concatenates their byte content and yields the
concatenation as a byte string.

Examples:

~~~
$ edn-abnf -abytes -e 'bytes<<>>'
h''
$ edn-abnf -abytes -e 'bytes`text1`'
h'7465787431'
$ edn-abnf -abytes -e 'bytes<<"1", "2">>'
h'3132'
$ edn-abnf -abytes -e 'bytes<<"ä", h'"'2f'"'>>'
h'C3A42F'
$ edn-abnf -abytes -e 'bytes<<"ä", h'"'2f'"'>>' | diag2diag.rb -tu
'ä/'
~~~
{: post="fold"}

## float: IEEE754-oriented literals for more floating point values

The "`float`" application extension enables the notation of 2-byte,
4-byte, and 8-byte byte strings to express floating point values
(mt=7, ai=25/26/27 respectively) by giving their IEEE 754
representation.
A text string used as an argument is interpreted exactly as a hex
literal (like the `h` application prefix); the result is used as the
byte string.

The application-oriented literal is interpreted as an encoded data
item would be that prefixes the byte string by a single byte 0xF9
(2 bytes, i.e., binary16), 0xFA (4 bytes, i.e., binary32), and 0xFB (8
bytes, i.e., binary64), respectively.
Byte strings of a different length than 2, 4, or 8 raise an error.

Example:

~~~
$ edn-abnf -afloat -e "[float'fe00', float'47110815']" -tpretty
82             # array(2)
   F9 FE00     # primitive(65024)
   FA 47110815 # primitive(1192298517)
$ edn-abnf -afloat,same -e "same<< float'47110815', 0x1.22102ap+15 >>"
37128.08203125
~~~
{: post="fold"}

The purpose of this application extension is to close a gap in EDN's
{{IEEE754}} binary64 support:
Without this (or a similar) extension there is no way to represent NaN
values different from the one called out at the end of {{Section 4.1 of
RFC8949@-cbor}}: "(for many applications, the single NaN encoding
0xf97e00 will suffice)".
For finite floating point numbers, the decimal or hex floating point
representations are preferred.

## tbd b32/h32/c32?: Create byte string from base32 representation

[^todo] define, possibly beyond the two variants defined in {{-base}}; watch {{-basebis}}.

## tbd b45: Create byte string from base45 representation

[^todo] define based on {{-base45}}

Implementation Status
=====================
{: removeinrfc}

<!-- RFC7942 -->

{::boilerplate RFC7942}

## Implementation 1

The "`float`" application extension is implemented in
[](https://cbor.me) and can be enabled (`–‍afloat`) in the cbor
diagnostic tools (`cbor-diag` gem) and the `edn-abnf` gem.\\
At the time of writing, the "`same`" and "`bytes`" application
extensions  (`–‍asame`,  `–‍abytes`) are only available in the gems.

## Implementation 2

The `float` application extension is implemented in the JavaScript
tools that come with the CBOR test vectors project [](https://github.com/cbor-wg/cbor-test-vectors/blob/main/check/files.test.js).

Security considerations
=======================

The security considerations of {{-edn}} apply.

[^todo] List any specific security considerations that apply to
specific application extensions.

IANA considerations
===================

[^todo]

--- back

{::include-all lists.md}

Acknowledgements
================
{: unnumbered}


[^note]: Note:
[^chore]: Chore:
[^todo]: TO BE DONE:
[^issue]: Open issue:
[^rfced]: RFC Editor:

[^discuss]: Discuss:
