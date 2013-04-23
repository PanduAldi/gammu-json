The `gammu-json` utility
========================

A command-line interface to the important portions of
[`libgammu`](https://github.com/gammu/gammu). Send and receive text
messages programmatically using a USB GSM or CDMA modem. Speaks JSON
and UTF-8 by default. Minimal dependence on external libraries.

This package is usable now, but still under active development.
Please refrain from using it in production applications until we're
sure that we've shaken all the bugs out. For more information, please
review the [documentation for Gammu](http://wammu.eu/gammu/) directly.

If you want to **skip all of this summary-level stuff and go straight to
[the examples](#examples)**, we won't be offended.

Summary
-------

A simple command-line utility to send, retrieve, and delete SMS messages using
`libgammu` and a supported phone and/or GSM/CDMA modem. In all cases, input is
accepted as UTF-8 encoded program arguments, and output is provided as UTF-8
encoded JSON.

Multi-part messages (sometimes referred to as "concatenated SMS") are
supported.

In send mode, messages are automatically broken up in to multiple message parts
when size limits require it. User-data headers (UDHs) are generated
automatically to allow for message reassembly.

When sending messages, the utility automatically detects whether the message
can be sent using the default GSM alphabet (described in GSM 03.38). If so,
message parts may be up to 160 characters in length. If any character in the
string cannot be represented in the default GSM alphabet, the *entire message*
will be encoded as UCS-2, reducing the number of characters available per
message (or message part). This is a limitation of the underlying SMS protocol;
it cannot be fixed without making changes to the protocol itself.

In receive mode, message parts are each returned as separate messages; however,
every part of a multipart message will have the `udh`, `segment`, and
`total_segments` attributes set to meaningful values. The `udh` value is
a unique 8-bit or 16-bit integer, generated by the sender, that differentiates
sets of concatenated messages from one-another -- that is, messages with the
same `udh` value should be linked together.

This utility does not perform message reassembly in receive mode; reassembly
requires some form of persistent data storage, since each message part could be
delayed indefinitely. Architecturally speaking, message reassembly will need to
be performed at a higher level.

Building
--------

Building `gammu-json` from source can typically be accomplished by running
`make` and `make install`. The included `Makefile` uses `pkg-config` to locate
`libgammu`. If you'd like to link with a `libgammu` that isn't under `/usr`,
try defining the `PREFIX` variable when you run `make`, like this:

```
make PREFIX=/srv/software/libgammu
```

or, alternatively, set the `PKG_CONFIG_PATH` environment variable to point
directly to your preferred `$prefix/lib/pkgconfig` directory.

Setup
-----

By default, `gammu-json` requires that (a) you have a valid `/etc/gammurc`
file (or symbolic link) present in your local filesystem, and (b) that
`/etc/gammurc` provides a working phone/modem configuration in its `gammu`
section. Here's a simple example:

```
[gammu]
Connection = at
Device = /dev/ttyUSB0
```

If your gammu configuration file is located elsewhere, you may use the
`-c` or `--config` option to specify a custom configuration file path.

Licensing
---------

This software is released under the
[GNU General Public License (GPL) v3](http://www.gnu.org/licenses/gpl-3.0.txt).

Executing and/or parsing the output of an unmodified `gammu-json` from your
closed-source program *does not* create a derivative work of `gammu-json`, nor
does it require you to release whatever source code it is that you're afraid of
other people seeing. Your secret's safe with us.

However: if you modify the `gammu-json` software itself, and distribute a
compiled version containing your modifications to others (either as software,
as a software package, or as software preloaded on a piece of hardware), you
will be required to offer those modifications (in their original human-readable
source code form) to whomever obtains the modified software.

Please note that Gammu itself (including the `libgammu` upon which `gammu-json`
relies) is licensed under the GPL v2 (or later, presumably at your option).

Examples
--------

### Usage

Note: JSON output is reformatted here (and in all other examples) to improve
readability. If you'd like the results of `gammu-json` to be automatically
indented (i.e. "pretty-printed") for your application, you can pipe its output
to your favorite formatting utility.

```shell
$ gammu-json
```
```
Usage:
  gammu-json { retrieve | send { phone text }... | delete N... }
```


### Sending (simple)

Sending a single message is easy.

```shell
$ gammu-json send '+15035551212' 'This is a simple test message.'
```
```json
[
  {
    "parts_sent": 1,
    "index": 1,
    "parts_total": 1,
    "parts": [
     {
      "index": 1,
      "reference": 250,
      "status": 0,
      "content": "This is a simple test message.",
      "result": "success"
     }
    ],
    "result": "success"
  }
]
```

### Sending (multiple messages)

Sending more than one message is also easy.

```shell
$ gammu-json send \
  '+15035551212' 'This is a simple test message.' \
  '+15035551212' 'This is another simple test message.'
```
```json
[
  {
    "parts_sent": 1,
    "index": 1,
    "parts_total": 1,
    "parts": [
     {
      "index": 1,
      "reference": 250,
      "status": 0,
      "content": "This is a simple test message.",
      "result": "success"
     }
    ]
  },
  {
    "parts_sent": 1,
    "index": 2,
    "parts_total": 1,
    "parts": [
     {
      "index": 1,
      "reference": 251,
      "status": 0,
      "content": "This is another simple test message.",
      "result": "success"
     }
    ],
    "result": "success"
  }
]
```

### Sending (multipart concatenated messages)

A message that is too long for a single SMS (160 characters for the 7-bit GSM
default alphabet, or 80 for UCS-2 coding of Unicode symbols) will be split in
to a concatenated/multipart message automatically. Information about how the
message was split will be returned in the JSON output (see the _parts_ array).

```shell
$ gammu-json send '+15035551212' 'This is a simple test message. This is only a test. Had this been an actual message, the authorities in your area (with cooperation from federal and state authorities) would have already read it for you.'
```
```json
[
  {
    "parts_sent": 2,
    "index": 1,
    "parts_total": 2,
    "parts": [
     {
      "index": 1,
      "reference": 251,
      "status": 0,
      "content": "This is a simple test message. This is only a test. Had this been an actual message, the authorities in your area (with cooperation from federal and stat",
      "result": "success"
     },
     {
      "index": 2,
      "reference": 252,
      "status": 0,
      "content": "e authorities) would have already read it for you.",
      "result": "success"
     }
    ],
    "result": "success"
  }
]
```

### Sending (Unicode characters in UTF-8 or UCS-2)

If a message contains any UTF-8 character that is not present in the 7-bit
default GSM alphabet, the message will automatically be sent as a two byte per
character UCS-2 SMS.

```shell
$ gammu-json send '+15035551212' 'This is a test message. الحروف عربية. ان شاء الله.'
```
```json
[
  {
    "parts_sent": 1,
    "index": 1,
    "parts_total": 1,
    "parts": [
       {
          "index": 1,
          "reference": 254,
          "status": 0,
          "content": "This is a test message. الحروف عربية. ان شاء الله.",
          "result": "success"
       }
    ],
    "result": "success"
  }
]
```

### Sending (multipart UCS-2 messages)

For UCS-2 messages, messages will be sent in multiple parts after only 80
characters (rather than the usual limit of 160). To see the UCS-2 message size
limitation in action, try this example again after removing the obvious
non-Latin characters.


```shell
$ gammu-json send '+15035551212' 'The portion before this contains only Latin characters. Nepali text follows this. हो'
```
```json
[
 {
  "parts_sent": 2,
  "index": 1,
  "parts_total": 2,
  "parts": [
     {
        "index": 1,
        "reference": 2,
        "status": 0,
        "content": "The portion before this contains only Latin characters. Nepali text",
        "result": "success"
     },
     {
        "index": 2,
        "reference": 3,
        "status": 0,
        "content": " follows this. हो",
        "result": "success"
     }
  ],
  "result": "success"
 }
]
```

### Retrieval (empty)

Retrieving messages from a newly-purchased SMS modem yields the empty JSON
array, and exits with zero status. The program will display an error message
on `stderr` and exit with a non-zero status if something goes wrong.

```shell
$ gammu-json retrieve
```
```json
[]
```

### Retrieval (simple)

After running the `retrieve` command, a JSON-encoded array of message objects
is returned on `stdout`. The phone number for the sender and "short message
service center" (SMSC) are each included, along with a receive timestamp, an
SMSC receive timestamp (if available), location number (on the SMS modem), user
data header (UDH) value, and segment information (in this case, one of one).

```shell
$ gammu-json retrieve
```
```json
[
 {
  "location" : 1,
  "smsc" : "+12085552222",
  "content" : "This is a test message.",
  "segment" : 1,
  "inbox" : true,
  "smsc_timestamp" : false,
  "folder" : 1,
  "udh" : false,
  "timestamp" : "2013-04-02 17:05:49",
  "from" : "+15035551212",
  "total_segments" : 1,
  "encoding" : "utf-8"
 },
 {
  "location" : 2,
  "smsc" : "+12085552222",
  "content" : "This is another test message.",
  "segment" : 1,
  "inbox" : true,
  "smsc_timestamp" : false,
  "folder" : 1,
  "udh" : false,
  "timestamp" : "2013-04-02 17:06:03",
  "from" : "+15155551111",
  "total_segments" : 1,
  "encoding" : "utf-8"
 }
]

```
### Retrieval (multipart messages)

Multipart messages are returned in multiple segments, tied together by
the sender's phone number in `from`, and the user data header value in
`udh`. Single-part messages will have a `udh` value of false. Multipart
messages that are _missing_ the proper header information will have a
`udh` value of `null`.

```shell
$ gammu-json retrieve
```
```json
[
 {
  "location" : 1,
  "smsc" : "+12085032222",
  "content" : "This is a simple test message. This is only a test. Had this been an actual message, the authorities in your area (with cooperation from federal and stat",
  "segment" : 1,
  "inbox" : true,
  "smsc_timestamp" : false,
  "folder" : 1,
  "udh" : 215,
  "timestamp" : "2013-04-02 17:12:40",
  "from" : "+15035551212",
  "total_segments" : 3,
  "encoding" : "utf-8"
 },
 {
  "location" : 2,
  "smsc" : "+12085032222",
  "content" : "e authorities) would have already read it for you. This is a simple test message. This is only a test. Had this been an actual message, the authorities i",
  "segment" : 2,
  "inbox" : true,
  "smsc_timestamp" : false,
  "folder" : 1,
  "udh" : 215,
  "timestamp" : "2013-04-02 17:12:52",
  "from" : "+15035551212",
  "total_segments" : 3,
  "encoding" : "utf-8"
 },
 {
  "location" : 3,
  "smsc" : "+12085032222",
  "content" : "n your area (with cooperation from federal and state authorities) would have already read it for you.This is a simple test message. This is only a test.",
  "segment" : 3,
  "inbox" : true,
  "smsc_timestamp" : false,
  "folder" : 1,
  "udh" : 215,
  "timestamp" : "2013-04-02 17:12:59",
  "from" : "+15035551212",
  "total_segments" : 3,
  "encoding" : "utf-8"
 },
 {
  "location" : 4,
  "smsc" : "+12085032222",
  "content" : "This is a short message.",
  "segment" : 1,
  "inbox" : true,
  "smsc_timestamp" : false,
  "folder" : 1,
  "udh" : false,
  "timestamp" : "2013-04-02 17:13:56",
  "from" : "+15035551212",
  "total_segments" : 1,
  "encoding" : "utf-8"
 }
]
```
### Deletion (simple)

This example assumes there are seven messages stored on the SMS modem,
numbered one through seven.

```shell
$ gammu-json delete all
```
```json
{
 "detail" : {
   "1" : "ok",
   "2" : "ok",
   "3" : "ok",
   "4" : "ok",
   "5" : "ok",
   "6" : "ok",
   "7" : "ok"
   },
 "result" : "success",
 "totals" : {
   "errors" : 0,
   "requested" : "all",
   "deleted" : 7,
   "attempted" : 7,
   "examined" : 7,
   "skipped" : 0
  }
}
```
### Deletion (selective)

This example assumes that there are twelve messages (or message segments),
numbered one through twelve.  Message numbers are one-based integer
identifiers, and are returned in the `gammu-json retrieve` output as the
`location` property. Currently, `gammu-json` always deletes from folder zero,
which contains all available messages on the phone/modem.

```shell
$ gammu-json delete 3 1 4 5 9
```
```json
{
 "detail" : {
  "1" : "ok",
  "2" : "skip",
  "3" : "ok",
  "4" : "ok",
  "5" : "ok",
  "6" : "skip",
  "7" : "skip",
  "8" : "skip",
  "9" : "ok",
  "10" : "skip",
  "11" : "skip",
  "12" : "skip"
 },
 "result" : "success",
 "totals" : {
  "errors" : 0,
  "requested" : 5,
  "deleted" : 5,
  "attempted" : 5,
  "examined" : 12,
  "skipped" : 7
 }
}
```

### Deletion (of non-existent messages)

This example assumes that there are four messages (or message segments),
numbered one to four. Deleting non-existent messages is not an error;
rather the nonexistent messages are noted in the `totals.attempted`
property, and the `result` of the deletion is reported as `partial`.

```json
{
   "detail" : {
      "1" : "ok",
      "2" : "ok",
      "3" : "ok",
      "4" : "skip"
   },
   "result" : "partial",
   "totals" : {
      "errors" : 0,
      "requested" : 6,
      "deleted" : 3,
      "attempted" : 3,
      "examined" : 4,
      "skipped" : 1
   }
}
```

Authors
-------

Copyright © 2013 David Brown ``<hello at scri.pt>``
<br />
Copyright © 2013 Medic Mobile, Inc. ``<david at medicmobile.org>``

All rights reserved. Meticulously handcrafted with love in Portland, Oregon, USA.

Legal
-----

The above copyright notice and this permission notice shall be included
in all copies or substantial portions of the Software.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS
IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO,
THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR
PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL DAVID BROWN BE LIABLE FOR ANY
DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS
OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION)
HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT,
STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN
ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
POSSIBILITY OF SUCH DAMAGE.



