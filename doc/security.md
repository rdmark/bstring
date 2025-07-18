Better String library Security Statement
========================================

by Paul Hsieh

Introduction
------------

The Better String library (hereafter referred to as Bstrlib) is an attempt to
provide improved string processing functionality to the C and C++ languages.
At the heart of the Bstrlib is the management of "bstring"s which are a
significant improvement over '\0' terminated char buffers. See the
accompanying documenation file bstrlib.txt for more information.

DISCLAIMER: THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND
CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT
NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A
PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR
CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL,
EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO,
PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS;
OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY,
WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR
OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF
ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

Like any software, there is always a possibility of failure due to a flawed
implementation.  Nevertheless a good faith effort has been made to minimize
such flaws in Bstrlib.  Use of Bstrlib by itself will not make an application
secure or free from implementation failures, however, it is the author's
conviction that use of Bstrlib can greatly facilitate the creation of
software meeting the highest possible standards of security.

Part of the reason why this document has been created, is for the purpose of
security auditing, or the creation of further "Statements on Security" for
software that is created that uses Bstrlib. An auditor may check the claims
below against Bstrlib, and use this as a basis for analysis of software which
uses Bstrlib.

Statement on Security
---------------------

This is a document intended to give consumers of the Better String Library
who are interested in security an idea of where the Better String Library
stands on various security issues. Any deviation observed in the actual
library itself from the descriptions below should be considered an
implementation error, not a design flaw.

This statement is not an analytical proof of correctness or an outline of one
but rather an assertion similar to a scientific claim or hypothesis. By use,
testing and open independent examination (otherwise known as scientific
falsifiability), the credibility of the claims made below can rise to the
level of an established theory.

Common Security Issues
----------------------

### Buffer Overflows

The Bstrlib API allows the programmer a way to deal with strings without
having to deal with the buffers containing them. Ordinary usage of the
Bstrlib API itself makes buffer overflows impossible.

Furthermore, the Bstrlib API has a superset of basic string functionality as
compared to the C library's `char *` functions, C++'s `std::string` class and
Microsoft's MFC based `CString` class. It also has abstracted mechanisms for
dealing with IO. This is important as it gives developers a way of migrating
all their code from a functionality point of view.

### Memory Size Overflow/Wrap Around Attack

By design, Bstrlib is impervious to memory size overflow attacks.  The
reason is that it detects length overflows and leads to a result error before
the operation attempts to proceed.  Attempted conversions of char* strings
which may have lengths greater than INT_MAX are detected and the conversion
is aborted.  If the memory to hold the string exceeds the available memory
for it, again, the result is aborted without changing the prior state of the
strings.

### Constant String Protection

Bstrlib implements runtime enforced constant and read-only string semantics,
i.e., bstrings which are declared as constant via the `bsStatic` macro cannot
be modified or deallocated directly through the Bstrlib API, and this cannot
be subverted by casting or other type coercion.

### Aliased Bstring Support

Bstrlib detects and supports aliased parameter management throughout the API.
The kind of aliasing that is allowed is the one where pointers of the same
basic type may be pointing to overlapping objects (this is the assumption the
ANSI C99 specification makes). Each function behaves as if all read-only
parameters were copied to temporaries which are used in their stead before
the function is enacted (it rarely actually does this). No function in the
Bstrlib uses the `restrict` parameter attribute from the ANSI C99
specification.

### Information leaking

In bstraux.h, using the semantically equivalent macros `bSecureDestroy` and
`bSecureWriteProtect` in place of `bdestroy` and `bwriteprotect`
respectively will ensure that stale data does not linger in the heap's free
space after strings have been released back to memory. Created bstrings are
not linked to anything external to themselves, and thus cannot expose
deterministic data leaking. If a bstring is resized, the preimage may exist
as a copy that is released to the heap. Thus for sensitive data, the bstring
should be sufficiently presized before manipulated so that it is not
resized.  `bSecureInput` has been supplied in bstraux.c, which can be used
to obtain input securely without any risk of leaving any part of the input
image in the heap except for the allocated bstring that is returned.

### Memory leaking

Bstrlib does not do anything out of the ordinary to attempt to deal with the
standard problem of memory leaking (losing references to allocated memory)
when programming in the C and C++ languages. However, it does not compound
the problem any more than exists either, as it doesn't have any intrinsic
inescapable leaks in it. Bstrlib does not preclude the use of automatic
garbage collection mechanisms such as the Boehm garbage collector.

### Encryption

Bstrlib does not present any built-in encryption mechanism. However, it
supports full binary contents in its data buffers, so any standard block
based encryption mechanism can make direct use of bstrings for buffer
management.

### Double freeing

Freeing a pointer that is already free is an extremely rare, but nevertheless
a potentially ruthlessly corrupting operation (its possible to cause Win 98 to
reboot, by calling free mulitiple times on already freed data using the WATCOM
CRT).  Bstrlib invalidates the bstring header data before freeing, so that in
many cases a double free will be detected and an error will be reported
(though this behaviour is not guaranteed and should not be relied on).

Using `bstrFree` pervasively (instead of `bdestroy`) can lead to somewhat
improved invalid `free` avoidance (it is completely safe whenever bstring
instances are only stored in unique variables). For example:

    struct tagbstring hw = bsStatic("Hello, world");
    bstring cpHw = bstrcpy(&hw);

    #ifdef NOT_QUITE_AS_SAFE
    bdestroy(cpHw); /* Never fail */
    bdestroy(cpHw); /* Error sometimes detected at runtime */
    bdestroy(&hw);  /* Error detected at run time */
    #else
    bstrFree(cpHw); /* Never fail */
    bstrFree(cpHw); /* Will do nothing */
    bstrFree(&hw);  /* Will lead to a compile time error */
    #endif

### Resource Based Denial of Service

`bSecureInput` has been supplied in bstraux.c. It has an optional upper limit
for input length. But unlike `fgets`, it is also easily determined if the
buffer has been truncated early. In this way, a program can set an upper limit
on input sizes while still allowing for implementing context specific
truncation semantics, i.e., does the program consume but dump the extra
input, or does it consume it in later inputs?

### Mixing `char *`s and Bstrings

The bstring and `char *` representations are not identical. So there is a risk
when converting back and forth that data may lost. Essentially bstrings can
contain `'\0'` as a valid non-terminating character, while `char *` strings
cannot and in fact must use the character as a terminator. The risk of data
loss is very low, since:

1. The simple method of only using bstrings in a `char *` semantically
   compatible way is both easy to achieve and pervasively supported.

2. Obtaining `'\0'` content in a string is either deliberate or indicative
   of another, likely more serious problem in the code.

3. The library comes with various functions which deal with this issue
   (namely: `bfromcstr`, `bstr2cstr`, and `bSetCstrChar`)

Marginal Security Issues
------------------------

### 8-bit Versus 9-bit Portability

Bstrlib uses `CHAR_BIT` and other limits.h constants to the maximum extent
possible to avoid portability problems. However, Bstrlib has not been tested
on any system that does not represent char as 8-bits. So whether or not it
works on 9-bit systems is an open question. It is recommended that Bstrlib be
carefully auditted by anyone using a system in which `CHAR_BIT` is not 8.

### EBCDIC/ASCII/UTF-8 Data Representation Attacks

Bstrlib uses ctype.h functions to ensure that it remains portable to non-
ASCII systems. It also checks range to make sure it is well defined even for
data that ANSI does not define for the ctype functions.

Obscure Issues
--------------

### Data attributes

There is no support for a Perl-like "taint" attribute, although this is a
fairly straightforward exercise using C++'s type system.
