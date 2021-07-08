# Serialized Data Expression Format
Version: v0.0.3 Pre-Draft  
Date: July 7, 2021  
License: CC BY 4.0  
Author: Dave Huseby <dwh@trustframe.com>  
Copyright: (c) TrustFrame, Inc. 2021  

## Data Expressions

There are only two forms that these expressions may take: regular and type-define. The regular form is an s-expression:

```sexp=
(<label> <data>)
```

Expressions of this form are serialized data and the regular form is recursive in that the data may be one or more s-expressions.

The type-define form is for defining concrete types of structured data. The type-define form starts with a left parenthesis and a single quote followed by a string, a space, and one or more s-expressions or a single regex string defining the structure of the data. It ends with a right parenthesis: 

```sexp=
('<type name> <s-expr>+|<regex>)
```

When defining the new type with `('` the data for the type can either be a single regular expression or it can be one or more s-expressions that bind types to named members of the type. This is all that is needed to define arbitrary data structures as well as how they are parsed from text.

Each type definition gets added to the global list of types as aliases for the regular expression in the type regex. The type regex may reference other, previously defined types so that the regex is built up recursively.

Type definitions and regular data expressions can be in the same file. The only restriction is that a type definition must appear before any data expressions referencing the type occur.

## Extensibility

To make SDEF fully extensible, the only other special form needed is an expression to include other files at parsing time. This is accomplished with a special "include expression" that is a type-define expression with no type name that names the location of an SDEF file to include and process. An include expression defines one or more URL's for files to parse and process at the point of the include expression. This allows for reuse of type definitions and to also include data from other files.

As an example, I may extend my own local type definitions with a standard one for comments like so:

```sexp=
(' https://trustframe.com/sdef/comment.sdef)

("A single hexidecimal character")
('hex_char [0-9a-fA-F])

("A hex encoded key is 128 hex chars")
('KeyHex hex_char{128})

("A list of one or more keys")
('Keys KeyHex+)
```

The `('` include tells the parser to download and process the `comment.sdef` file before proceeding. The `comment.sdef` file defines just the comment type. By including the comment type definition, my local type definition file can have comments in it anywhere. For the record, the definition of a comment type is:

```sexp=
('" (?:[^"]*)(?:\"))
```

This says that any expression starting with the double quotation mark `"` should match all characters that are not a double quotation mark--including newlines--and then match an ending double quotation mark. They are defined as non-matching groups in the regex because we want to ignore the data in a comment. The result of processing any comment is "nil", or void, which causes the parser to ignore it. Thus comments can be placed anywhere in a serialized data file.

Another point that the comment type brings up is that type labels are matched using a greeding matching against the list of type labels known to the parser. This is so that there is no requirement for a space between the label and the data in a data expression. This allows comment data expressions to be of the form `("this is a comment")`. If there are two types with the labels "A" and "Ab", then a data expression such as `(About)` is matched with the type "Ab" and the remainder of the data will be checked against the regular expression from the "Ab" type definition.

If the type definitions for hexchar and KeyHex are in the file named `keyhex.sdef` stored at `http://trustframe.com/sdef` then an example of serialized data that defines a Keys list of KeyHex objects looks like the following:

```sexp=
(' http://trustframe.com/sdef/keyhex.sdef)

("My public keys")
(Keys
    1a87a6fa65a9663c4192fe74ccb88a51bb1b2251d7041a858512f970bf1230d2454617014478c93a6b319dfc7f9353d9bcd46c5d031d2e2be3339170971a09c3
    454617014478c93a6b319dfc7f9353d9bcd46c5d031d2e2be3339170971a09c31a87a6fa65a9663c4192fe74ccb88a51bb1b2251d7041a858512f970bf1230d2
)
```

## Complex Data Types

Defining complex data types is as simple as nesting named expressions under a newly defined type. Let's assume I want to define a Person type that includes one or more email addresses and one or more phone numbers. First I have to define the email and phone number types:

```sexp=
(' http://trustframe.com/sdef/comment.sdef)

("An email requires a crazy regex")
('email (?:[a-z0-9!#$%&'*+/=?^_`{|}~-]+(?:\.[a-z0-9!#$%&'*+/=?^_`{|}~-]+)*|"(?:[\x01-\x08\x0b\x0c\x0e-\x1f\x21\x23-\x5b\x5d-\x7f]|\\[\x01-\x09\x0b\x0c\x0e-\x7f])*")@(?:(?:[a-z0-9](?:[a-z0-9-]*[a-z0-9])?\.)+[a-z0-9](?:[a-z0-9-]*[a-z0-9])?|\[(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?|[a-z0-9-]*[a-z0-9]:(?:[\x01-\x08\x0b\x0c\x0e-\x1f\x21-\x5a\x53-\x7f]|\\[\x01-\x09\x0b\x0c\x0e-\x7f])+)\]))

("A phone number is 10 digits, no spaces")
('phone [0-9]{10})
```

Assume that the phone and email types are stored in a file called `pfields.sdef`. Then the definition of a `Person` type could look like:

```
(' http://trustframe.com/sdef/pfields.sdef)

("A person has one or more emails followed by one or more phones")
('Person
    (emails email+)
    (phones phone+)
)
```

With an example of a serialized Person object looks like this:

```
(' http://foo.se/person.sdef)
(Person
    (emails
        joe@schmoe.com
        joe@foo.com
    )
    (phones
        9119119111
        8005551212
    )
)
```

It is important to note a few things about the definition of complex data objects. First of all in the `('Person` definition, instead of giving a regex directly, the definition includes two s-expressions that bind names to types. This is how named fields of a type are defined. So in the case above, each `Person` has two fields, one named `emails` and the other named `phones`. Furthermore, `emails` is defined as a list of one or more `email` typed data and `phones` is defined as a list of one or more `phone` typed data.

When defining a type as a list of another named type, the data type for each member of the list is inferred from the type definition and does not require explicit usage in data. It is important to note that all data units in a list are of the same type to simplify parsing. The follow is also a valid encoding of the above `Person` object but is pedantic and not necessary:

```sexp=
(' http://trustframe.com/sdef/person.sdef)
(Person
    (emails
        (email joe@schmoe.com)
        (email joe@foo.com)
    )
    (phones
        (phone 9119119111)
        (phone 8005551212)
    )
)
```

The explicit encoding of each email and phone number object is unnecessary because the parser generated from the type definition knows that all objects in `emails` are `email` typed. Same goes for the list of phone numbers. The extra encoding is implied and redundant.

#### Field Ordering

One other aspect of the type definition process is that the order of the fields is significant. Unlike JSON, the field ordering in the type definitions elliminates the need for an externally defined and arbitrary canonicalization algorithm. All `Person` objects have a list of `emails` and then a list of `phones`.

If you defined another type, say `AntiPerson` that had a list of `phones` before a list of `emails`, the two types are considered distinct because the order of their fields is different.

This design detail is important because SDEF must have a single, trivial way to canonicalize an SDEF file before creating or verifying a digital signature. It also enables automated parser generation and completes the self-describing attribute of this encoding scheme.

#### Type Names

By convention, top-level or first-order objects are named with camel case names. In the examples above, objects like `Person` and `HexKey` are top-level types. Names for basic types should be short and are lower case with underbars separating words. In the examples above, types such as `email` and `phone` and `hex_char` are basic types. Also by convention, field names in complex types are also all lower case with underbars separating words.

### Canonicalization

To canonicalize any SDEF encoded object, the only thing we need to worry about is normalizing whitespace. The rules for normalizing whitespace are simple. We need to preserve one space between expression parameters unless they are themselves an s-expression. The following is also a valid serialization of the `Person` object above:

```sexp=
(' http://trustframe.com/sdef/person.sdef)(Person (emails joe@schmoe.com joe@foo.com)(phones 9119119111 8005551212))
```

Note that there is no space between s-expressions but there is a space between the email addresses and the phone numbers. Technically the space between an expression type identifier and the expression parameters is not necessary but we add one for readability. The only exception is if the regex for a given type parameter is a non-matching group such as the definition for comments. The regex is non-matching so we don't put a space after `("` which also increases readability. 

The above normalized and serialized form of the Person object is used when sending data over a wire or when doing some cryptography such as calculating a digital signature over a piece of data.

## Digital Signatures

One feature that many serialization formats need to tackle is how to include digital signatures over data. JSON has to use a combination of application-specific rules (e.g. signature is included last--see Secure ScuttleButt format) plus a complicated canonicalization algorithm to get JSON objects into some normalized form for the digital signature creation and verification to work. The unconstrained field ordering in JSON is a perpetual problem for canonicalizing JSON files for cryptrographic operations.

With data in serialized data expression format (SDEF), signatures are just another data type and included somewhere in a data object. By convention they should probably always be last in an object just to make both creating and verifying digital signatures easier for implementors.

If the [CDE](https://hackmd.io/@dhuseby/BkG-4gp2u) encoding format is chosen for encoded cryptographic constructs, such as keys and signatures, the task of defining keys and signatures for inclusion into SDEF data is trivial.

First of all, a digitally signed object is some data followed immediately by its signature. Let us define a "Signed" type that is either a digitally signed email or a digitally signed phone number.

```sexp=
("A digitally signed email or phone number")
('Signed email|phone signature)
```

This relies on the email and phone types defined in previous examples. It also uses a new type called "signature". In the CDE encoding standard, all digital signatures begin with either a lower-case or upper-case letter S. Further more, all CDE encoded data is encoded using the ASCII characters a-z, A-Z, 0-9, hyphen (`-`) and underscore (`_`). The definition of the signature type is therefore trivial:

```sexp=
("A CDE format digital signature")
('signature [sS][a-zA-Z0-9-_]+)
```

It is just a lower or upper-case letter S followed by one or more of the CDE encoding characters. This works because CDE formatted digital signatures are self-describing and the algorithm used is encoded along with the signature.

Because of the way CDE works, implementors can limit the kinds of digital signatures to one or more specific algorithms. For instance, all standard Ed25519 digital signatures being with "se" and all OpenPGP digital signatures begin with "sp" so to limit valid signatures to those types, just define the regex for a signature to start with either of those:

```sexp=
("Either an OpenPGP or Ed25519 signature")
('signature s[ep][a-zA-Z0-9-_]+)
```

## Standard Types

To further simplify and standardize the use of SDEF, a set of standard basic types shoudl be defined. Types such as hexidecimal strings, Base64/Base58/etc strings, URI strings, Email strings, etc are probably all worth defining so that others don't have to. The hope is that we can settle on some standard regular expressions for basic types that everybody uses to increase reuse and simplicity.

```
('" (?:[^"]*)(?:\"))

("hexidecimal characters for hex strings")
    ('hex [0-1a-fA-F])
    
("A URI string")
    ('uri \w+:(\/?\/?)[^\s]+)

("An email address")
    ('email (?:[a-z0-9!#$%&'*+/=?^_`{|}~-]+(?:\.[a-z0-9!#$%&'*+/=?^_`{|}~-]+)*|"(?:[\x01-\x08\x0b\x0c\x0e-\x1f\x21\x23-\x5b\x5d-\x7f]|\\[\x01-\x09\x0b\x0c\x0e-\x7f])*")@(?:(?:[a-z0-9](?:[a-z0-9-]*[a-z0-9])?\.)+[a-z0-9](?:[a-z0-9-]*[a-z0-9])?|\[(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?|[a-z0-9-]*[a-z0-9]:(?:[\x01-\x08\x0b\x0c\x0e-\x1f\x21-\x5a\x53-\x7f]|\\[\x01-\x09\x0b\x0c\x0e-\x7f])+)\]))

("Base64 encoded data")
    (?:[A-Za-z0-9+/]{4})*(?:[A-Za-z0-9+/]{2}==|[A-Za-z0-9+/]{3}=)?
```

## Change Log
* v0.0.1, July 21, 2019 -- Initial version specifying a strict ordering, text based, self-describing data structure serialization format for data that is intended to be digitally signed.
* v0.0.2, July 7, 2020 -- Some small fixes with the addition of mentions to the Universal Cryptographic Construct encoding method for including binary cryptographic data.