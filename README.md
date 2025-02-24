# ksuid [![Go Report Card](https://goreportcard.com/badge/github.com/segmentio/ksuid)](https://goreportcard.com/report/github.com/segmentio/ksuid) [![GoDoc](https://godoc.org/github.com/segmentio/ksuid?status.svg)](https://godoc.org/github.com/segmentio/ksuid) [![Circle CI](https://circleci.com/gh/segmentio/ksuid.svg?style=shield)](https://circleci.com/gh/segmentio/ksuid.svg?style=shield)

ksuid is an efficient, comprehensive, battle-tested Go library for
generating and parsing a specific kind of globally unique identifier
called a *KSUID*. This library serves as its reference implementation.

## Install
```sh
go get -u github.com/segmentio/ksuid
```

## What is a KSUID?

KSUID is for K-Sortable Unique IDentifier. It is a kind of globally
unique identifier similar to a [RFC 4122 UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier), built from the ground-up to be "naturally"
sorted by generation timestamp without any special type-aware logic.

In short, running a set of KSUIDs through the UNIX `sort` command will result
in a list ordered by generation time.

## Why use KSUIDs?

There are numerous methods for generating unique identifiers, so why KSUID?

1. Naturally ordered by generation time
2. Collision-free, coordination-free, dependency-free
3. Highly portable representations

Even if only one of these properties are important to you, KSUID is a great
choice! :) Many projects chose to use KSUIDs *just* because the text 
representation is copy-and-paste friendly.

### 1. Naturally Ordered By Generation Time

Unlike the more ubiquitous UUIDv4, a KSUID contains a timestamp component
that allows them to be loosely sorted by generation time. This is not a strong
guarantee (an invariant) as it depends on wall clocks, but is still incredibly
useful in practice. Both the binary and text representations will sort by 
creation time without any special sorting logic.

### 2. Collision-free, Coordination-free, Dependency-free

While RFC 4122 UUIDv1s *do* include a time component, there aren't enough
bytes of randomness to provide strong protection against collisions
(duplicates). With such a low amount of entropy, it is feasible for a
malicious party to guess generated IDs, creating a problem for systems whose
security is, implicitly or explicitly, sensitive to an adversary guessing
identifiers.

To fit into a 64-bit number space, [Snowflake IDs](https://blog.twitter.com/2010/announcing-snowflake)
and its derivatives require coordination to avoid collisions, which
significantly increases the deployment complexity and operational burden.

A KSUID includes 128 bits of pseudorandom data ("entropy"). This number space
is 64 times larger than the 122 bits used by the well-accepted RFC 4122 UUIDv4
standard. The additional timestamp component can be considered "bonus entropy"
which further decreases the probability of collisions, to the point of physical
infeasibility in any practical implementation.

### Highly Portable Representations

The text *and* binary representations are lexicographically sortable, which
allows them to be dropped into systems which do not natively support KSUIDs
and retain their time-ordered property.

The text representation is an alphanumeric base62 encoding, so it "fits"
anywhere alphanumeric strings are accepted. No delimiters are used, so
stringified KSUIDs won't be inadvertently truncated or tokenized when
interpreted by software that is designed for human-readable text, a common
problem for the text representation of RFC 4122 UUIDs.

## How do KSUIDs work?

Binary KSUIDs are 24-bytes: a 64-bit unsigned integer UTC timestamp (nanoseconds),
and a 128-bit randomly generated payload. The timestamp uses big-endian
encoding, to support lexicographic sorting. The timestamp epoch is adjusted
to May 13th, 2014, providing over 300 years of life. The payload is
generated by a cryptographically-strong pseudorandom number generator.

The text representation is always 33 characters, encoded in alphanumeric
base62 that will lexicographically sort by timestamp.

## High Performance

This library is designed to be used in code paths that are performance
critical. Its code has been tuned to eliminate all non-essential
overhead. The `KSUID` type is derived from a fixed-size array, which
eliminates the additional reference chasing and allocation involved in
a variable-width type.

The API provides an interface for use in code paths which are sensitive
to allocation. For example, the `Append` method can be used to parse the
text representation and replace the contents of a `KSUID` value
without additional heap allocation.

All public package level "pure" functions are concurrency-safe, protected
by a global mutex. For hot loops that generate a large amount of KSUIDs
from a single Goroutine, the `Sequence` type is provided to elide the
potential contention.

By default, out of an abundance of caution, the cryptographically-secure
PRNG is used to generate the random bits of a KSUID. This can be relaxed
in extremely performance-critical code using the included `FastRander`
type. `FastRander` uses the standard PRNG with a seed generated by the
cryptographically-secure PRNG.

*_NOTE:_ While there is no evidence that `FastRander` will increase the
probability of a collision, it shouldn't be used in scenarios where
uniqueness is important to security, as there is an increased chance
the generated IDs can be predicted by an adversary.*

## Battle Tested

This code has been used in production at Segment for several years,
across a diverse array of projects. Trillions upon trillions of
KSUIDs have been generated in some of Segment's most
performance-critical, large-scale distributed systems.

## Plays Well With Others

Designed to be integrated with other libraries, the `KSUID` type
implements many standard library interfaces, including:

* `Stringer`
* `database/sql.Scanner` and `database/sql/driver.Valuer`
* `encoding.BinaryMarshal` and `encoding.BinaryUnmarshal`
* `encoding.TextMarshal` and `encoding.TextUnmarshal`
  (`encoding/json` friendly!)

## Command Line Tool

This package comes with a command-line tool `ksuid`, useful for
generating KSUIDs as well as inspecting the internal components of
existing KSUIDs. Machine-friendly output is provided for scripting
use cases.

Given a Go build environment, it can be installed with the command:

```sh
$ go install github.com/segmentio/ksuid/cmd/ksuid
```

## CLI Usage Examples

### Generate a KSUID

```sh
$ ksuid
021V4cKaCMv7F2pKzGdyg78BaLq036UK5
```

### Generate 4 KSUIDs

```sh
$ ksuid -n 4
021V4d3hRmKTBAzkpBPWc82nBKeUyX2S8
021V4d3hRmKSqPy3ZCWCReKT5F5vmLTnt
021V4d3hRmKT4V2JgLtiCHfYCDNl1Xpd4
021V4d3hRmKSFL5y2L2CJ5Ip9WdSbLSyb
```

### Inspect the components of a KSUID

```sh
$ ksuid -f inspect 021V4cKaCMv7F2pKzGdyg78BaLq036UK5

REPRESENTATION:

  String: 021V4cKaCMv7F2pKzGdyg78BaLq036UK5
     Raw: 0306ACB45225E5883D1907A4D6BC78F423EBF166583D09D5

COMPONENTS:

       Time: 2021-04-10 12:45:22.4463538 +0200 CEST
  Timestamp: 218051522446353800
    Payload: 3D1907A4D6BC78F423EBF166583D09D5
```

### Generate a KSUID and inspect its components 

```sh
$ ksuid -f inspect

REPRESENTATION:

  String: 021V51f1SRLcImQnBTO50EOsa38fAZxBg
     Raw: 0306ACDF508FDCE48883F409F91DF7A6CD28D87BAAB478A8

COMPONENTS:

       Time: 2021-04-10 12:48:27.1033377 +0200 CEST
  Timestamp: 218051707103337700
    Payload: 8883F409F91DF7A6CD28D87BAAB478A8

```

### Inspect a KSUID with template formatted inspection output

```sh
$ ksuid -f template -t '{{ .Time }}: {{ .Payload }}' 021V4d3hRmKTBAzkpBPWc82nBKeUyX2S8
2021-04-10 12:45:27.751335 +0200 CEST: 3005CA0E5674346CECACB33E0AE8AFD0
```

### Inspect multiple KSUIDs with template formatted output

```sh
$ ksuid -f template -t '{{ .Time }}: {{ .Payload }}' $(ksuid -n 4)
2021-04-10 12:49:46.3138813 +0200 CEST: 325A114EA96F9DC1AAF6C47EE14ECD1B
2021-04-10 12:49:46.3138813 +0200 CEST: 9665DE19740E29D612BD9D16E885A226
2021-04-10 12:49:46.3138813 +0200 CEST: EA35FAE54CA3C33A235402315F7EE2F6
2021-04-10 12:49:46.3138813 +0200 CEST: 8DE009CF2DC941444F6C400F6F5A8A51
```

### Generate KSUIDs and output JSON using template formatting

```sh
$ ksuid -f template -t '{ "timestamp": "{{ .Timestamp }}", "payload": "{{ .Payload }}", "ksuid": "{{.String}}"}' -n 4
{ "timestamp": "218052156732343200", "payload": "E1F34D98AD45DF36DB549D3CD249E7E9", "ksuid": "021V61Kx47L4R92UMPrfTpT9czrHkmJZZ"}
{ "timestamp": "218052157580340000", "payload": "6D074D776529AD19E2F5BD3E86913DEA", "ksuid": "021V61SACfBGSy5XWQLx64uOhRdrDeqrK"}
{ "timestamp": "218052158214601500", "payload": "52AB4701827913ED037E98F7D044E67F", "ksuid": "021V61XYdofwdiYuM8tX2czh4B93RXo6x"}
{ "timestamp": "218052159132931200", "payload": "B64066A1EB8BAC93E7CEE725DAD6D565", "ksuid": "021V61fMrfhiqPU4fZhoQljqH0XKfv6qj"}
```

## License

ksuid source code is available under an MIT [License](/LICENSE.md).
