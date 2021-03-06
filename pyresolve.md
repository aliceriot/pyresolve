#Resolving DNS queries with Python

I read a nice [short cartoon](https://howdns.works/) about how DNS works,
and I wanted to understand it in more detail. I thought it would be a nice
learning exercise to write a DNS resolver in Python - we're not going to
performance here, but instead for clean readable code and clear
explanations.

##Literate Python stuff

Before we get into the code let's explain this whole literate Python thing
a bit. I decided to use the [Pweave](https://github.com/mpastell/Pweave)
module, since it has the most options for output formats (.md, .tex,
.html, .rst, etc) and also lets you write your document's markup in
a variety of languages.

If you clone the repo to your computer you can do a `pip install -r
requirements.txt` to get all the dependencies for this project (including
Pweave). I'd recommend installing Pweave globally (rather than in
a virtualenv) because I ran into some issues when trying to use it in
a virtualenv.

Then if you want to actually run the program you can do:

```bash
Ptangle pyresolve.mdw
```

which will 'tangle' the code. This terminology comes from Knuth's original
literate programming setup, which was called WEB and consisted of LaTeX
markup and Pascal source code. 'Weaving' a WEB document is taking this
combined source and producing a cleanly typeset document, and 'tangling'
the WEB is extracting just the source code from it. If you're reading this
on Github you're actually reading the output of:

```bash
Pweave -f pandoc pyresolve.mdw
```

So we 'weave' the source to get out a nice clean .md file. Tangling should
give you a `pyresolve.py` which you can run to test it out. I did not make
any attempts to test for Python 2 compatibility, so it may not work with
that. I am quite sure things are working on Python 3, however! 

A note: Vim will open the file in Markdown mode (because it has the .md
extension) but this may not be what you want - I would rather have syntax
highlighting and plugins for the Python code when editing Python. So I did
these two bindings in my `~/.vimrc`:

```
nnoremap <Leader>lp :setlocal ft=python<cr>
nnoremap <Leader>md :setlocal ft=markdown<cr>
```

which works really smoothly!

###Tests

A side effect of all this is you need to Tangle the source in order to run
the tests. There's a little shellscript called `tests.sh` that does this
for you. If you've already installed Pweave globally it should work!

##DNS queries

Anyway, how does DNS work? Well, it turns out it is sort of complicated!
The DNS is mainly laid out in RFCs
[1034](https://www.ietf.org/rfc/rfc1034.txt) and
[1035](https://www.ietf.org/rfc/rfc1035.txt), which (although very
informative) are quite long. 

The first bit of information we need is in 1035, in section 4 (Messages).
This is how the packet we're going to build for our DNS query is laid out:

```
+---------------------+
|        Header       |
+---------------------+
|       Question      | the question for the name server
+---------------------+
|        Answer       | RRs answering the question
+---------------------+
|      Authority      | RRs pointing toward an authority
+---------------------+
|      Additional     | RRs holding additional information
+---------------------+
```

(the IETF's ASCII art skills are legendary!). Basically, I think we`re
going to try to do this as some sort of bytestring, which Python gives us
lots of tools to work with.

But first, some import statements!


~~~~{.python}
import random
import sys
~~~~~~~~~~~~~



Alright, so basically the packet is going to contain a bunch of different
sections, each of which we'll handle as a different object.

###Header

The header is the first part of our query. It holds some information about
what we are asking the server for, what kind of question, etc. The IETF
kindly supplies us with more ASCII perfection:

```
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                      ID                       |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|QR|   Opcode  |AA|TC|RD|RA|   Z    |   RCODE   |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                    QDCOUNT                    |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                    ANCOUNT                    |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                    NSCOUNT                    |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                    ARCOUNT                    |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```

Great, we're going to make a new class to hold all this info! Here is what
we need to do:

1. **ID** This is a 16 bit identifier that we assign when we generate
  a query, and the responder will copy it onto our reply. This lets us
  match the response to the query that generated it, useful if we are
  firing off lots of them!

2. **QR**: this is a one-bit field which specifies whether this is a query
   (0) or a response (1). We want 0!

3. **OPCODE**: this identifies the query type. We just want standard query,
   which means we want to use the value 0

4. **AA**: this is 'authoritative answer', which will get changed in the
   response.

5. **TC**: may change to 1 in the response to indicate that the response
   was truncated (due to size constraints)

6. **RD**: indicates we'd like the server to resolve our query
   recursively.

7. **RA**: the server can set this to indicate if recursion support is
   available (we'll leave it 0)

8. **Z**: this is reserved! For magical future use. We'll leave it set to
   zero.

9. **RCODE**: this gets set by the server when it's formulating its
   response. 0 indicates success, 1-5 indicate various kinds of failures
   or errors.

10. **QDCOUNT**: 16 bits to specify how many questions we want to ask 
    (this is passed in as 'numq').

11. **ANCOUNT**: another 16 bits to specify how many resource records
    we'll put the in the answer section (for the server to write into).

12. **NSCOUNT**: yet another 16 bits, this time for the number of records
    stored in the authority records section.

13. **ARCOUNT**: 16 more bits, to store the number of resource records in
    the 'additional records' section.

Whew! That was a lot. Most of these are zero though, which helps. We also
are going to make one more class attribute, which is `self.header`. This
will be equal to the 'genheader' method, which sticks everything together
for us. Since we need to generate a packet (which is binary data) we will 
going to use 'pack', from the struct module:


~~~~{.python}
from struct import pack
~~~~~~~~~~~~~



Here is what that class looks like:


~~~~{.python}
class Header(object):
    def __init__(self, numq):
        self.id = random.choice(range(0, 65535))
        self.qr = 0b0
        self.opcode = 0b0
        self.aa = 0b0
        self.tc = 0b0
        self.rd = 0b0
        self.ra = 0b0
        self.z = 0b000
        self.rcode = 0b0000
        self.qdcount = numq
        self.ancount = numq
        self.nscount = 0b0
        self.arcount = 0b0
        if numq not in range(0, 65535) or type(numq) == float:
            raise ValueError("number of queries out of range")
        self.header = self.genheader()

    def genheader(self):
        """
        generate the header bytearray
        the only tricky part is the 2nd line
        """
        secondline = (self.qr << 15) | (self.opcode << 11) | \
            (self.aa << 10) | (self.tc << 9) | (self.rd << 8) | \
            (self.ra << 7) | (self.z << 4) | (self.rcode)

        return pack('HHHHHH', self.id, secondline, self.qdcount,
                self.ancount, self.nscount, self.arcount)
~~~~~~~~~~~~~



All we want out of this `Header` class is a nice clean way to generate the
bytearray for the header section, so we do not have too much going on
here. We do some bit shifting to get the class variables into the right
order. The `pack` function is a nifty little thing: we supply a format
string (`HHHHHH` above) and a number of variables, and it produces a byte
array with those variables formatting according to the format string. `H`
tells pack to put in a 'short' integer, which takes up two bytes. 

Great! So once we have a header that looks nice we can move on to working
on the questions section, which is the next part of our query packet.

###Questions

This is the next section of our packet, which actually holds the questions
that we want to ask the DNS server. Great! More ASCII art fun:

```
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                                               |
/                     QNAME                     /
/                                               /
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                     QTYPE                     |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                     QCLASS                    |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```


