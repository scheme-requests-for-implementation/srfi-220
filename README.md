# SRFI nnn: Line directives

by Lassi Kortela

## Status

Draft

## Abstract

Many programming tools rely on specially formatted source code
comments to annotate code with language-agnostic metadata. Such "magic
comments" are hard for both humans and computers to parse reliably, as
the purpose of a comment is to be free-form text that is not
interpreted by machine.

This SRFI extends the standard Scheme directive syntax (`#!`) to
support _line directives_. They look like magic comments to
language-agnostic tools but read as S-expressions in Scheme, combining
the portability of magic comments with the well-defined syntax and
easy parsing of ordinary Scheme code.

## Issues

## Rationale

### Survey of #! syntax in Lisp standards

#### Common Lisp

The ANSI Common Lisp standard, section [2.4.8 Syntax -> Standard Macro
Characters -> Sharpsign](http://clhs.lisp.se/Body/02_dh.htm), says the
dispatch macro character `#!` is explicitly reserved to the user. No
conforming implementation defines it.

#### ISLisp

The [ISLisp
standard](https://web.archive.org/web/20160313021746/http://islisp.info/Documents/PDF/islisp-2007-03-17-pd-v23.pdf)
(2007) uses `#` as a dispatch macro character like Common Lisp and
Scheme. It does not appear to use `#!` for anything.

#### Scheme

Scheme standards treat `#!` as a magic marker generally followed by a
directive that looks like an identifier but serves a special purpose
distinct from an ordinary identifier with the same spelling.

* R<sup>2</sup>RS uses `#!true` and `#!false` for booleans and
  `#!null` for the empty list.
* R<sup>3</sup>RS does not use `#!`.
* R<sup>4</sup>RS does not use `#!`.
* R<sup>5</sup>RS does not use `#!`.
* R<sup>6</sup>RS uses `#!r6rs` and permits any other identifier.
* R<sup>7</sup>RS uses `#!fold-case` and `#!no-fold-case`.

DSSSL and SRFI 89 use `#!key` and `#!optional` and `#!rest` for lambda
list markers.

Various Scheme implementations use other identifiers following `#!`
for [other purposes](https://registry.scheme.org/#hash-bang-syntax).

SRFI 22 (_Running Scheme Scripts on Unix_) recognizes `#!` at the very
beginning of the file as starting a comment that runs until the end of
the line.

Guile uses `#!...!#` anywhere in the file for multi-line comments.

### Survey of #! syntax in non-standard Lisp dialects

#### AutoLisp

`#` is a constituent character in symbols. `#!` does not appear to be
used for anything.

#### Clojure and ClojureScript

`#!` **unofficially** begins a comment that runs until the end of the
line.

Source: In the Clojure codebase, open
`src/jvm/clojure/lang/LispReader.java` and look for
`dispatchMacros['!'] = new CommentReader();`.

#### Emacs Lisp

`#!` **unofficially** begins a comment that runs until the end of the
line.

Source: In the GNU Emacs codebase, open `src/lread.c` and look inside
`read1()`. The comment in the source code says `#! appears at the
beginning of an executable file. Skip the first line.` but the code
actually skips `#!` anywhere in the file.

#### Fennel

`#!` at the very **beginning of the file** starts a comment that runs
until the end of the line.

Source: In the Fennel codebase, open `src/fennel/parser.fnl` and look
inside `string-stream`. It replaces a `#!` at the start of the file
with a traditional `;;` comment prefix.

#### Hy

`#` begins a comment that runs until the end of the line. By
extension, `#!` does the same.

#### Janet

`#` **officially** begins a comment that runs until the end of the
line. By extension, `#!` does the same.

#### Lisp Flavored Erlang

`#` is a constituent character in symbols. Otherwise `#` does not
appear to be used for anything, and causes an `illegal character`
syntax error.

#### NewLisp

`#` **officially** begins a comment that runs until the end of the
line. By extension, `#!` does the same.

#### PicoLisp

`#` **officially** begins a comment that runs until the end of the
line. By extension, `#!` does the same.

As a minor exception, `#` does not begin a comment when it is part of
a symbol.

#### Rep (librep)

`#!...!#` at the very **beginning of the file** is treated like a
comment.

Elsewhere in the file `#!key` and `#!optional` and `#!rest` are read
in as symbols, and `#!` followed by anything else is a syntax error.

Source: In the `librep` codebase, open `src/lisp.c` and look inside
`readl()`.

### Conclusion

Both major Lisp standards, Common Lisp and Scheme, are easily amenable
to adding a `#!` syntax. While Common Lisp reserves `#!` for users,
customizing the readtable is quite rare in practice, and the pervasive
use of `#!` in Unix scripts makes it unlikely that Common Lisp users
would configure `#!` to do anything other than read a comment.

Of the surveyed non-standard Lisp dialects, all except Fennel and Rep
handle `#!` anywhere in the file akin to a comment. Both Fennel and
Rep handle `#!` at the beginning of the file as a comment, which means
their syntax could easily be amended to handle `#!` elsewhere in the
file the same way.

Guile and Rep use `#!...!#` comments, i.e. they require a `!#` at the
end. We can expect `!#` to cause a read error in most Scheme
implementations. However, using `;!#` causes it to be treated as an
ordinary comment that runs until the end of the line in those Schemes.

## Specification

Scheme's read syntax is extended such that the two-character sequence
`#!` followed immediately by either a newline, the end of the file, or
one or more horizontal whitespace charcters, causes a line directive
to be read as follows:

1. Start accumulating elements into an empty list.
2. Skip whitespace and comments.
3. If at end of line or no longer on the original line, return.
4. Read one form and accumulate at the end of the list.
5. If at end of line or no longer on the original line, return.
6. Go to step 2.

The accumulated list is returned as the Scheme representation of the
line directive. The list is not evaluated as Scheme code, but may be
either ignored or passed to an implementation-defined facility for
processing.

### Recursive line directives

It is an error to write things like `#! #! foo` or `#! outer (#!
inner)`.

### Equivalence of `#! r6rs` with a space and `#!r6rs` without

It is implementation-defined whether `#! r6rs` (with whitespace), once
read, is processed the same way or in a different way compared to
`#!r6rs` (without whitespace).

### Symbols and keywords

Some Scheme implementations have read syntax for _keywords_. These
syntaxes exist:

* `foo:`
* `:foo`
* `#:foo`

It also varies whether or not keywords are disjoint from symbols.

TODO: Should this SRFI specify that keywords are normalized into
symbols?

### Extending the formal syntax of R7RS

R7RS section `7.1. Formal syntax` add a new case to the definition of
`<directive>`:

    #! <intraline whitespace>+ <directive datums>

where

    <directive datums> = | <datum> <intraline whitespace>* <directive datums>

and `<datum>` in this case contains no newlines in the datum or
surrounding whitespace.

## Examples

For clarity, all Scheme symbols in the examples are written using the
`|vertical-bar|` syntax from R7RS.

### Unix script interpreter

Directive:

    #! /usr/bin/env fantastic-scheme

Read as S-expression:

    (|/usr/bin/env| |fantastic-scheme|)

### License information

Directives:

    #! Copyright © 2019, 2020 Just A. Schemer <schemer@example.org>
    #! SPDX-License-Identifier: GPL-3.0-or-later

Read as S-expressions:

    (|Copyright| |©| 2019 (|unquote| 2020)
     |Just| |A.| |Schemer| |<schemer@example.org>|)
    (|SPDX-License-Identifier:| |GPL-3.0-or-later|)

### Text editor settings for Emacs

Directive:

    #! -*- mode: scheme -*-

Read as S-expression:

    (|-*-| |mode:| |scheme| |-*-|)

Directives:

    #! Local Variables:
    #! mode: scheme
    #! coding: utf-8
    #! comment-column: 0
    #! End:

Read as S-expressions:

    (|Local| |Variables:|)
    (|mode:| |scheme|)
    (|coding:| |utf-8|)
    (|comment-column:| 0)
    (|End:|)

### Text editor settings for Vim

Directive:

    #! vim: ft=lisp tw=60 ts=2 expandtab fileencoding=euc-jp :

Read as S-expression:

    (|vim:| |ft=lisp| |tw=60| |ts=2| |expandtab| |fileencoding=euc-jp| |:|)

### Text editor settings for both Emacs and Vim

Directive:

    #! -*- mode: scheme -*- vim: set ft=scheme :

Read as S-expression:

    (|-*-| |mode:| |scheme| |-*-| |vim:| |set| |ft=scheme| |:|)

## Implementation

## Acknowledgements

## References

## Copyright

Copyright (C) Lassi Kortela (2021).

Permission is hereby granted, free of charge, to any person obtaining
a copy of this software and associated documentation files (the
"Software"), to deal in the Software without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Software, and to
permit persons to whom the Software is furnished to do so, subject to
the following conditions:

The above copyright notice and this permission notice shall be
included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE
LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION
OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION
WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
