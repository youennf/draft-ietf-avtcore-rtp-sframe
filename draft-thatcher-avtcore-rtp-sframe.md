---
title: "RTP Payload Format for SFrame"
docname: draft-thatcher-avtcore-rtp-sframe-latest
category: std
date: {DATE}

ipr: trust200902
area: "Applications and Real-Time"
workgroup: "Audio/Video Transport Core Maintenance"
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]
submissiontype: IETF

author:
 -
    ins: P. Thatcher
    name: Peter Thatcher
    organization: Microsoft
    email: pthatcher@microsoft.com

--- abstract

This document describes the RTP payload format of SFrame.

--- middle

# Introduction

SFrame {{?I-D.draft-ietf-sframe-enc-01}} describes an end-to-end encryption and authentication mechanism
for media frames in a multiparty conference call, in which central media servers (SFUs) can access the
media metadata needed to make forwarding decisions without having access to the actual media.

This document describes how to packetize a media frame encrypted using SFrame into RTP packets.

# Terminology and Notation

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.


# RTP Packetization of a media frame encrypted by SFrame

In order to packetize SFrame into RTP, packetization is done in 2 stages.
In the first stage, before SFrame encryption, media is packetized into RTP packets in a way specific to the media format.
In the second stage, each RTP packet from the first stage is packetized into RTP packets in a way specific to SFrame.
SFrame encryption is applied to the payload of each RTP packet between the first and second stages.

For example, if a media frame to be encrypted by SFrame is encoded using VP8, the media frame is first
packetized according to {{!RFC7741}} into one RTP packets with VP8-specific payloads.  Each of those
VP8 RTP payloads are then encrypted using SFrame, resulting in an SFrame-encrypted RTP payload of VP8.
SFrame-specific packetization is then applied to the SFrame-encrypted RTP payload of VP8, resulting in
RTP packets with SFrame-specific RTP payloads.

SFrame-specific packetization is done by first breaking up the output of SFrame encryption
into fragments, and then prepending some fragment metadata necessary for depacketization.  Finally,
fragments are combined with values from the RTP header of the output of the media-format-specific
packetization.

The SFrame-specific RTP payloads (fragments with prepended metadata) have the following format:

~~~
 0                   1                   2
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|L| media PT    |  media frame ID               |
| fragment index                |  fragment ... |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

The media PT must be the payload type of the output of the media-format-specific packetization.
The frame index of the first fragment of each media frame MUST be 0.
The frame index of each subsequent fragment MUST be one more than the previous fragment.
The L bit MUST be 0 for all fragments except for the last one of the media frame.
The media frame ID must be unique enough that a depacketizer may be able to differentiate
the fragments of one media frame from another.
The SSRC, timestamp, marker bit, CSRCs, and header extensions of the SFrame RTP packets MUST be the same
as those of the output of the media-format-specific packetization.
The payload type of the SFrame RTP packets must be a payload type that indicates the payload
format defined in this document, and it must have a negotiated RTP clock rate that is the same as the
media-format-specific RTP packet.

# RTP depacketization of SFrame

Depacketization is done by doing the packetization process in reverse:

1. The fragments of a given media frame ID are grouped together in order of fragment index and concatenated together, resulting in a media frame encrypted by SFrame.

2. The media frame is decrypted using SFrame, resulting in a media-format-specific RTP payload.

3. The media-format-specific RTP payload is combined with the RTP headers of the RTP packet with fragment index 0, resulting in a media-format-specific RTP packet.
   The "media PT" from the SFrame RTP payload header is used as the payload type of the media-format-specific RTP packet.

4. The media-format-specific RTP packet is passed into a media-format-specific RTP depacketizer, resulting in a media frame.


# SFrame payload type negotiation

Because the payload type of an RTP packet that results from SFrame-specific packetization must match the
clock rate of the payload type of the RTP packet that results from media-format-specific packetization,
it may be necessary to negotiate more than one SFrame payload type.  For example, if one were to use SDP
to negotiate payload types, the following payload types could be negotiated with different clock rates:

~~~
m=audio 50000 RTP/SAVPF 96
a=rtpmap:96 sframe/48000
m=video 50002 RTP/SAVPF 97
a=rtpmap:97 sframe/90000
~~~

# Security Considerations

This document is subject to the security considerations of SFrame.

# IANA Considerations

None
