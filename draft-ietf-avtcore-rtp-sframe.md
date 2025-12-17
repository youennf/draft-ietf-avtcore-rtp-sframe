---
title: "SFrame usage with RTP and SDP"
docname: draft-ietf-avtcore-rtp-sframe-latest
category: std
date: {DATE}

ipr: trust200902
area: "Applications and Real-Time"
workgroup: "Audio/Video Transport Core Maintenance"
keyword: Internet-Draft
venue:
  group: "Audio/Video Transport Core Maintenance"
  type: "Working Group"
  mail: "avt@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/avt/"
  github: "ietf-wg-avtcore/draft-ietf-avtcore-rtp-sframe"
  latest: "https://ietf-wg-avtcore.github.io/draft-ietf-avtcore-rtp-sframe/draft-ietf-avtcore-rtp-sframe.html"

stand_alone: yes
pi: [toc, sortrefs, symrefs]
submissiontype: IETF

author:
 -
    ins: P. Thatcher
    name: Peter Thatcher
    organization: Microsoft
    email: pthatcher@microsoft.com
 -
    ins: Y. Fablet
    name: Youenn Fablet
    organization: Apple
    email: youenn@apple.com

normative:
  WebRTC_Encoded_Transform:
    author:
      org: World Wide Web Consortium
    title: WebRTC Encoded Transform
    date: 2025-05
    target: https://w3c.github.io/webrtc-encoded-transform/

--- abstract

This document describes the RTP payload format of SFrame, its use in SDP, and per stream key derivation.

--- middle

# Introduction

SFrame {{!RFC9605}} describes an end-to-end encryption and authentication mechanism
for media data in a multiparty conference call, in which central media servers (SFUs) can access the
media metadata needed to make forwarding decisions without having access to the actual media.

This document describes how to packetize a media frame encrypted using SFrame into RTP packets, how to
signal support in SDP, and how to derive per SSRC SFrame keys from a common key.

# Terminology and Notation

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# SFrame format

An SFrame ciphertext comprises a header and encrypted data.
The SFrame header has a size varying between 1 to 17 bytes.
The encrypted data can be of arbitrary length and is larger than the unencrypted data by a fixed overhead that depends on the encryption algorithm.
The overhead can be up to 16 bytes.

An SFrame ciphertext having an arbitrary long length, an application may decide to partition the data encrypted with SFrame
into pieces small enough that the SFrame ciphertext fits in a single RTP packet.
We call this per-packet SFrame.
This has the advantage of allowing to decrypt the content as soon as received.

An alternative is to encrypt the data, a media frame typically, and send the SFrame ciphertext over several RTP packets.
We call this per-frame SFrame.
This has the advantage of limiting the SFrame overhead, especially for video frames.
This alternative is also compatible with {{WebRTC_Encoded_Transform}}, which is important for backward compatibility of existing services.

The RTP format presented in this document supports both alternatives.

# RTP Header Usage

The general RTP payload format for SFrame is depicted below.

~~~
   0                   1                   2                   3
   0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |V=2|P|X|  CC   |M|     PT      |       sequence number         |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                           timestamp                           |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |           synchronization source (SSRC) identifier            |
  +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
  |            contributing source (CSRC) identifiers             |
  |                             ....                              |
  +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
  |S E x x x x x x|                                               |
  |                                                               |
  :                       SFrame payload                          :
  |                                                               |
  |                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                               :    OPTIONAL RTP padding       |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

The first byte of the RTP payload is the SFrame payload descriptor.

The S bit of the SFrame payload descriptor MUST be 0 for all fragments except for the first one of the SFrame frame.

The E bit of the SFrame payload descriptor MUST be 0 for all fragments except for the last one of the SFrame frame.

The 6 remaining bits of the SFrame payload descriptor are reserved for future use.

The payload type (PT) identifies the format of the media encrypted with SFrame.

The SSRC, timestamp, marker bit, and CSRCs of the SFrame RTP packets MUST be the same
as those of the output of the media-format-specific packetization.
The header extensions of the SFrame RTP packets SHOULD be the same
as those of the output of the media-format-specific packetization, but some may be omitted
if it is known that the omitted header extensions do not need to be duplicated on each SFrame RTP packet.

# RTP Packetization of SFrame

SFrame packets can be generated either from RTP media packets or from media frames as defined by {{WebRTC_Encoded_Transform}}.

For per-packet SFrame, the following processing is done, with a media frame as input:

1. Generate a group of RTP media packets from the media frame using a media-format-specific packetizer.
   The media-format-specific packetizer needs to be made aware of the SFrame overhead that happens to each RTP packet.
2. For each RTP packet of the group, encrypt its payload with SFrame.
3. Prepend to each RTP packet payload a SFrame payload descriptor with the S and E bits set to 1.
4. Send each RTP packet of the group.

For per-frame SFrame, the following processing is done, with a media frame as input:

1. Generate a SFrame ciphertext from the media frame data.
2. Fragment the SFrame ciphertext in a group of payloads so that RTP packets generated from them do not exceed the network maximum transmission unit size.
3. Prepend a zero byte as the SFrame payload descriptor to all payloads of the group.
4. Set the first bit S of the SFrame payload descriptor of the first packet to 1.
5. Set the second bit E of the SFrame payload descriptor of the last packet to 1.
6. Generate a group of RTP packets from the group of payloads, using the media frame to generate the RTP header, including RTP header extensions.
7. Send each RTP packet of the RTP packet group.

# RTP depacketization of SFrame

Reception of SFrame packets is done as follows:

1. The fragments of a given SFrame ciphertext are grouped together in order of the RTP sequence number,
   the first packet of the group having its S bit set to 1 and the last packet of the group having its E bit set to 1.
   All packets between the first and last need to be in the group.
2. Concatenate the payloads of all packets of the group to form the SFrame ciphertext.
3. Decrypt the SFrame ciphertext to obtain the media decrypted data.
4. If per-packet SFrame is being used, the following processing is done:
   1. Assert that the group of packets consists of a single packet.
   2. Set the media decrypted data as the payload of the packet and send the packet to the media-format-specific RTP depacketizer.
   3. If the depacketizer cannot generate a media frame yet, abort these steps. Otherwise, generate a media frame from the depacketizer.
5. If per-frame SFrame is being used, the following processing is done:
   1. Assert that the group of packets all have the same payload type.
   2. Extract the media metadata from the group of packets.
   3. Generate a media frame from the media decrypted data and the media metadata.
6. Send the media frame to the receiving pipeline.

# SFrame SDP negotiation

SFrame packetization is indicated via a new "a=sframe" SDP attribute defined in this specification.
This attribute is used at media level, it does not appear at session level.

The presence of the "a=sframe" attribute in a media section (in either an offer or an answer) indicates that the
endpoint is expecting to receive RTP packets encrypted with SFrame for that media section, as defined below.

Once each peer has verified that the other party expects to receive SFrame RTP packets, senders are expected to send SFrame encrypted RTP packets.
If one peer expects to use SFrame for a media section and identifies that the other peer does not support it, the peer
is expected to stop the transceiver associated with the media section, which will generate a zero port for that m-section.

When SFrame is in use for that media section, it will apply to the relevant media encodings defined for that media section.
This includes RTP payload types bound to media packetizers and media depacketizers as defined in {{!RFC7656}}, typically audio formats such as Opus and RTP video formats such as H264.
This notably includes RTP payload types representing {{WebRTC_Encoded_Transform}} [encoded video frame formats](https://w3c.github.io/webrtc-encoded-transform/#dom-rtcencodedvideoframe-data) and [encoded audio frame formats](https://w3c.github.io/webrtc-encoded-transform/#dom-rtcencodedaudioframe-data).

This does not include RTP-Based Redundancy mechanisms as defined in {{!RFC7656}}.
For instance, RTX defined in {{!RFC4588}} will retransmit SFrame-based packets.
Forward error correction formats as defined in {{!RFC5109}} will protect the encrypted content.
For Redundant Audio Data, known as RED, as defined in {{!RFC2198}}, a RED packetizer will take as input SFrame encrypted media data instead of unencrypted media data.

If BUNDLE is in use and the "a=sframe" attribute is present for a media section but not for another media section of the same BUNDLE,
payload types for media encodings that are relevant for SFrame MUST NOT be reused between the two media sections.

Questions:

1. Should we precise how RTX/FEC works with SFrame packetization? No impact AFAIK since RTX/FEC would work on packets (whether SFramed or not).
2. Is RED current proposal (transmit SFrame ciphertext blocks) good enough? An alternative is to have SFrame being applied on the entire RED packet payload.
3. Should we allow `a=sframe` at session level to mean that all media sections want sframe?

Here is an example of SFrame being negotiated for audio (opus and CN) and for video (H264 and VP8):

~~~
m=audio 50000 RTP/SAVPF 10 11
a=sframe
a=rtpmap:10 opus/48000/2
a=rtpmap:11 CN/8000

m=video 50002 RTP/SAVPF 100 101
a=sframe
a=rtpmap:100 H264/90000
a=rtpmap:101 VP8/90000
~~~

# SFrame SDP RTP Key Management

When SFrame is used with SDP and specifies RTP as the media transport, an additional key derivation step MAY be applied to produce a unique key per SDP m= line using the SSRC.
The resulting per SSRC stream key is used as the initial key in the session for that m= line and becomes the input to the SFrame `derive_key_salt` function.

This initial derivation step starts with the `base_key` of the session. Then, each SSRC stream involved in the SDP session, MUST perform this derivation step to produce
the initial SFrame `ssrc_key` for that stream in the SDP session. This step is performed using HMAC-based Key Derivation Function (HKDF) {{!RFC5869}} as follows:

~~~
ssrc_key = HKDF-Expand(HKDF-Extract(SSRC, base_key), "SFrame 1.0 RTP Stream", CipherSuite.Nh)
~~~

In the derivation of ssrc_key:

* The SSRC is encoded as a 4-byte big-endian byte sequence and passed as the salt value to HKDF-Extract.

* The same CipherSuite is used for this step as for the per packet / frame step the resulting key will be used with.

# SFrame Ratcheting with per SSRC keys

If ratcheting is used with this per SSRC key derivation algorithm, the per SSRC key derivation step MUST be done once at the start of the session, and then each subsequent
ratchet is done on the per SSRC specific keys using the SFrame ratcheting algorithm described in section 5.1 of {{!RFC9605}}.
In this per SSRC key derivation algorithm, the same algorithm defined in section 5.1 of {{!RFC9605}} is used to produce the KID from the key_generation and ratchet_steps inputs. This results in all streams using the same KID, but different keys and CTR values.

This results in a key tree that looks like the following for an offer that has three different SSRC's a, b, and c.

~~~
+--------+
|base_key|
+---┬----+ Initial
    |     Derivation       Ratchet 1         Ratchet 2         Ratchet N
    |    +----------+   +-------------+   +-------------+   +-------------+
    |--->|ssrc_key_a|-->|ssrc_key_a[1]|-->|ssrc_key_a[2]|-->|ssrc_key_a[N]|
    |    +----------+   +-------------+   +-------------+   +-------------+
    |    +----------+   +-------------+   +-------------+   +-------------+
    |--->|ssrc_key_b|-->|ssrc_key_b[1]|-->|ssrc_key_b[2]|-->|ssrc_key_b[N]|
    |    +----------+   +-------------+   +-------------+   +-------------+
    |    +----------+   +-------------+   +-------------+   +-------------+
    +--->|ssrc_key_c|-->|ssrc_key_c[1]|-->|ssrc_key_c[2]|-->|ssrc_key_c[N]|
         +----------+   +-------------+   +-------------+   +-------------+
~~~

# Security Considerations

This document is subject to the security considerations of SFrame.

# IANA Considerations

This document defines a new attribute name to be listed in the "attribute-name" registry of the "SDP Parameters" registry group, according to the following data:

Type: attribute

SDP Name: sframe

Usage Level: media

Mux Category: NORMAL

Reference: RFC XXXX
