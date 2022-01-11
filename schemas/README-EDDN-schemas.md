# EDDN Schemas Documentation

## Introduction

EDDN is a
[zermoq](https://zeromq.org/) service to allow players of the game
[Elite Dangerous](https://www.elitedangerous.com/), published
by [Frontier Developments](https://www.frontier.co.uk/), to upload game data so
that interested listeners can receive a copy.

EDDN accepts HTTP POST uploads in a defined format representing this game data
and then passes it on to any interested listeners.

---
## Sources

There are two sources of game data, both provided by the publisher of the game,
Frontier Developerments.  They are both explicitly approved for use by 
third-party software.

### Journal Files

On the PC version of the game, "Journal files" are written during any game
session. These are in newline-delimited JSON format, with each line
representing a single JSON object. Frontier Developments publishes
documentation for the various events in their
[Player Tools & API Discussions](https://forums.frontier.co.uk/forums/elite-api-and-tools/)
forum.

In general the documentation is made available in a file named something like:

    Journal_Manual-v<version>

as both a MicroSoft word `.doc` file, or a `.pdf` file.  Historically the 
use of `_` versus `-` in those filenames has varied.

Consult the latest of these for documentation on individual events.  
However, be aware that sometimes the documentation is in error, possibly due to
not having been updated after a game client change.

### Companion API (CAPI) data

Frontier Developments provides an API to retrieve certain game data, even 
without the game running.  Historically this was for use by its short-lived
iOS "Companion" app, and was only intended to be used by that app. There was no
public documentation, or even admission of its existence.

Eventually, after some enterprising players had snooped the connections and
figured out the login method and endpoints, Frontier Developments
[allowed general use of this](https://forums.frontier.co.uk/threads/open-letter-to-frontier-developments.218658/page-19#post-3371472)
.

Originally the API authentication required being supplied with the email and
password as used to login to the game (but at least this was over HTTPS).

In late 2018 Frontier switched the authentication to using an oAuth2 flow,
meaning players no longer need to supply their email and password to
third-party sites and clients.

As of October 2021 there has still never been any official documentation about
the available endpoints and how they work. There is some
[third-party documentation](https://github.com/Athanasius/fd-api/blob/main/docs/README.md)
by Athanasius.

It is *not* recommended to use CAPI data as the source as it's fraught with
additional issues.  EDMarketConnector does so in order to facilitate
obtaining data without the player needing to open the commodities screen.

#### Detecting CAPI data lag

When using the Companion API please be aware that the server that supplies this
data sometimes lags behind the game - usually by a few seconds, sometimes by
minutes. You MUST check in the data from the CAPI that the Cmdr is
docked, and that the station and system names match those
reported from the Journal before using the data for the commodity, outfitting
and shipyard schemas:

1. Retrieve the commander data from the `/profile` CAPI endpoint.
2. Check that `commander['docked']` is true.  If not, abort.
3. Retrieve the data from the `/market` and `/shipyard` CAPI endpoints.
4. Compare the system and station name from the CAPI market data,
   `["lastStarport"]["name"]` and `["lastSystem"]["name"]`,
   to that from the last `Docked` or `Location` journal event.  If either does
   not match then you MUST **abort**.  This likely indicates that the CAPI 
   data is lagging behind the game client state and thus should not be used.

---

## Uploading messages

### Send only live data to the live schemas
You MUST **NOT** send information from any non-live (e.g. alpha or beta)
version of the game to the main schemas on this URL.

You MAY send such to this URL so long as you append `/test` to the `$schemaRef`
value, e.g.

    "$schemaRef": "https://eddn.edcd.io/schemas/shipyard/2/test",

You MUST also utilise these test forms of the schemas when first testing your
code.

There might also be a beta.eddn.edcd.io, or dev.eddn.edcd.io, service
available from time to time as necessary, e.g. for testing new schemas or
changes to existing ones.  Ask on the `#eddn` channel of the EDCD Discord 
(see https://edcd.github.io/ for an invite link).

Alternatively you could attempt
[running your own test instance of EDDN](../docs/Running-this-software.md).

### Sending data
Messages sent to EDDN **MUST**:

- Use the URL: `https://eddn.edcd.io:4430/upload/`.  Note the use of 
  TLS-encrypted HTTPS.  A plain HTTP request will elicit a `400 Bad 
  Request` response.
- Use the HTTP 1.1 protocol.  HTTP/2 is not supported at this time.
- Use a **POST** request, with the body containing the EDDN message.  No
  query parameters in the URL are supported or necessary.

The body of an EDDN message is a JSON object in UTF-8 encoding.  You SHOULD 
set a `Content-Type` header of `applicaton/json`, and NOT any of:

* `application/x-www-form-urlencoded`
* `multipart/form-data`
* `text/plain`

For historical reasons URL form-encoded data *is* supported, **but this is 
deprecated and no new software should attempt this method**.  We 
purposefully do not further document the exact format for this.

You *MAY* use gzip compression on the body of the message, but it is not 
required.

You should be prepared to handle all scenarios where sending of a message 
fails:

1. Connection refused.
2. Connection timed out.
3. Other possible responses as documented in
   [Server responses](#server-responses).

Carefully consider whether you should queue a 'failed' message for later 
retry.  In particular, you should ensure that one 'bad' message does not 
block other messages from being successfully sent.

You **MUST** wait some reasonable time (minimum 1 minute) before retrying 
any failed message.

You **MUST NOT** retry any message that received a HTTP `400` or `426` code.
An exception can be made if, **and only if**, *you have manually verified that 
you have fixed the issues with it (i.e. updated the schema/version to a
currently supported one and adjusted the data to fit that schema/version).*

You **MAY** retry a message that initially received a `413` response (in 
the hopes that the EDDN service admins decided to increase the maximum 
allowed request size), but should not do so too quickly or in perpetuity.

In general:

- No data is better than bad data.
- *Delayed* good data is better than degrading the EDDN service for others.

### Format of uploaded messages
Each message is a JSON object in UTF-8 encoding containing the following
key+value pairs:

1. `$schemaRef` - Which schema (including version) this message is for.
2. `header` - Object containing mandatory information about the upload;
    1. `uploaderID` - a unique ID for the player uploading this data.  
       Don't worry about privacy, the EDDN service will hash this with a key
       that is regularly changed so no-one knows who an uploader is in-game.
    2. `softwareName` - an identifier for the software performing the upload.
    3. `softwareVersion` - The version of that software being used.

   Listeners MAY make decisions about whether to utilise the data in any
   message based on the combination of `softwareName` and `softwareVersion`.
   
   **DO not** add `gatewaytimestamp` yourself. The EDDN Gateway will add
   this and will overwrite any that you provide, so don't bother.
4. `message` - Object containing the data for this message. Consult the
   relevant README file within this documentation, e.g.
   [codexentry-README.md](./codexentry-README.md). There are some general
   guidelines [below](#contents-of-message).

For example, a shipyard message, version 2, might look like:

```JSON
{
  "$schemaRef": "https://eddn.edcd.io/schemas/shipyard/2",
  "header": {
    "uploaderID": "Bill",
    "softwareName": "My excellent app",
    "softwareVersion": "0.0.1"
  },
  "message": {
    "systemName": "Munfayl",
    "stationName": "Samson",
    "marketId": 128023552,
    "horizons": true,
    "timestamp": "2019-01-08T06:39:43Z",
    "ships": [
      "anaconda",
      "dolphin",
      "eagle",
      "ferdelance",
      "hauler",
      "krait_light",
      "krait_mkii",
      "mamba",
      "python",
      "sidewinder"
    ]
  }
}
```

### Contents of `message`
Every message MUST comply with the schema its `$schemaRef` value cites.  

Apart from short time windows during deployment of a new version the live 
EDDN service should always be using
[the schemas as present in the live branch](https://github.com/EDCD/EDDN/tree/live/schemas).
So, be sure you're checking the live versions and not, e.g. those in the 
`master` or other branches.

Each `message` object must have, at bare minimum:

1. `timestamp` - string date and time in ISO8601 format. Whilst this
   technically allows for any timezone to be cited you SHOULD provide this in
   UTC, aka 'Zulu Time' as in the example above. You MUST ensure that you are
   doing this properly. Do not claim 'Z' whilst actually using a local time
   that is offset from UTC.
   
   If you are only utilising Journal-sourced data then simply using the 
   value from there should be sufficient as the PC game client is meant to 
   always be correctly citing UTC for this.  Indeed it has been observed, 
   in the Odyssey 4.0.0.1002 client, that with the Windows clock behind UTC 
   by 21 seconds both the in-game UI clock *and* the Journal event 
   timestamps are still properly UTC to the nearest second.

   Listeners MAY make decisions on accepting data based on this time stamp,
   i.e. "too old".
2. At least one other key/value pair representing the data. In general there 
   will be much more than this. Consult the
   [schemas and their documentation](./).

Because the first versions of some schemas were defined when only the CAPI 
data was available, before Journal files existed, many of the key names chosen
in the schemas are based on the equivalent in CAPI data, not Journal events.
This means ouy MUST rename many of the keys from Journal events to match the
schemas.

EDDN is intended to transport generic data not specific to any particular Cmdr
and to reflect only the data that every player would see in-game in station 
services or the local map. To that end, uploading applications MUST ensure
that messages do not contain any Cmdr-specific data (other than "uploaderID",
the "horizons" flag, and the "odyssey" flag).

The individual schemas will instruct you on various elisions (removals) to 
be made to comply with this.

Some of these requirements are also enforced by the schemas, and some things
the schemas enforce might not be explicitly called out here.  So, **do**
check what you're sending against the relevant schema(s) when making any 
changes to your code.

It is also advisable to Watch this repository on GitHub so as to be aware 
of any changes to schemas.

### Server responses
There are three possible sources of HTTP responses when sending an upload 
to EDDN.

1. The reverse proxy that initially accepts the request.
2. The python `bottle` module that the Gateway uses to process the 
   forwarded requests.  This might object to a message before the actual 
   EDDN code gets to process it at all.
3. The actual EDDN Gateway code.

Once a message has cleared the EDDN Gateway then there is no mechanism for any 
further issue (such as a message being detected as a duplicate in the 
Monitor downstream of the Gateway) to be reported back to the sender.

To state the obvious, if there are no issues with a request then an HTTP 
200 response will be received by the sender.  The body of the response 
should be the string `OK`.

#### Reverse Proxy responses
In addition to generic "you typoed the URL" and other such "you just didn't 
make a valid request" responses you might experience the following:

1. `408` - `Request Timed Out` - the sender took too long to make/complete 
   its request and the reverse proxy rejected it as a result.
2. `503` - `Service Unavailable` - the EDDN Gateway process is either not 
   running, or not responding.
3. `400` - `Bad Request` - if you attempt to use plain HTTP, rather than
   HTTPS.

#### bottle responses
1. `413` - `Payload Too Large` - `bottle` enforces a maximum request size 
   and the request exceeds that.  As of 2022-01-07 the limit is 1MiB, and 
   pertains to the plain-text size, not after gzip compression if used.
   To verify the current limit check for the line that looks like:

      ```
      bottle.BaseRequest.MEMFILE_MAX = 1024 * 1024 # 1MiB, default is/was 100KiB
      ```
   
   in
   [src/eddn/Gateway.py](https://github.com/EDCD/EDDN/blob/live/src/eddn/Gateway.py),
   as added in
   [commit  0e80c76cb564771465f61825e694227dcc3be312](https://github.com/EDCD/EDDN/commit/0e80c76cb564771465f61825e694227dcc3be312).

#### EDDN Gateway responses
For all failures the response body will contain text that begins `FAIL: `.  Currently two different HTTP status codes are utilised:

1. `400` - `Bad Request` - This indicates something wrong with the request 
   body.  Possibly due to a format issue (compression, form encoding), or 
   the actual content of the EDDN message:
   1. `FAIL: zlib.error: <detail>` - A failure to decompress a message that 
      claimed to be compressed.

   2. `FAIL: Malformed Upload: <detail>` - the message appeared to be 
      form-encoded, but either the format was bad or there was no `data` 
      key.

   3. `FAIL: JSON parsing: <detail>` - the 
       message couldn't be parsed as valid JSON.  e.g.

   ```
   FAIL: JSON parsing: Expecting property name enclosed in double quotes: line 1 column 2 (char 1)
   ```
   
   4. `FAIL: Schema Validation: <detail>` - the message failed to validate 
      against the cited schema. e.g.
   
   ```
   FAIL: Schema Validation: [<ValidationError: "'StarPos' is a required property">]
   ```
   
   The exact detail will be very much dependent on both the schema the 
   message cited and the contents of the message that failed to pass the 
   validation.

   In particular, if the message contains a key that is tagged 'disallowed' in 
   the schema the response will look like:

   ```
   FAIL: Schema Validation: "[<ValidationError: "{'type': ['array', 'boolean', 'integer', 'number', 'null', 'object', 'string']} is not allowed for 'BadVALUE'">]"
   ```
   This is due to the use of a JSON schema stanza that says "don't allow 
   any valid type for the value of this key" to trigger the error for such
   disallowed keys.

   Note how the **value** for the disallowed key is cited, not the key *name*
   itself.  This is a limitation of how the `jsonschema` python module
   reports errors, and we are
   [hoping to improve this](https://github.com/EDCD/EDDN/issues/163).

2. `426` - `Upgrade Required` - This indicates that the cited schema, or 
   version thereof, is outdated.  The body of the response will be:

   ```
   FAIL: Oudated Schema: The schema you have used is no longer supported. Please check for an updated version of your application.
   ```
   The wording here is aimed at users of applications that send messages 
   over EDDN.  If you're the developer of such an application then 
   obviously you need to update your code to use a currently supported 
   schema and version thereof.


There shouldn't be any other variants of a 'FAIL' message.  If you find
any then please
[open an issue on GitHub](https://github.com/EDCD/EDDN/issues/new)
with as much detail as possible so that we can update this documentation.

## Receiving messages

EDDN provides a continuous stream of information from uploaders. To use this
data you'll need to connect to the stream using ZeroMQ (a library is probably
available for your language of choice).

The URL for the live Relay is:

    tcp://eddn.edcd.io:9500

Depending on the programming language and library used, you might need
to explicitly subscribe to an empty topic, `''`, in order to receive
anything.

Unfortunately at this time we're using an old version of
ZeroMQ which does not support server-side subscription filtering.  So by
all means specify a more specific topic filter if it suits your needs,
but realise that the dropping of not-matching messages will happen at
your end, not the server.

Once you've connected to that you will receive messages.  To access the 
data you will first need to zlib-decompress each message.  Then you will 
have a textual JSON object as per the schemas.

In general, check the guidance for [Uploading messages](#uploading-messages)
for the expected format of the messages.

Consumers can utilise the `$schemaRef` value to determine which schema a 
particular message is for.  There is no need to validate the messages 
against the schemas yourself, as that is performed on the EDDN Gateway.  
Messages that do not pass the schema validation there are not forwarded to 
receivers.

There is [example code](https://github.com/EDCD/EDDN/tree/master/examples)
available for a variety of programming languages to help you get started.