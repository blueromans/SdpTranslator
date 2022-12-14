## Introduction

(or _Unified Plan, Plan B and the answer to life, the universe and eveything!_)

If somebody wants to talk about interoperability between Firefox and Chome when
doing multi-party video conferences, it is impossible to not talk a little bit
(or a lot!) about [Unified
Plan](https://tools.ietf.org/html/draft-roach-mmusic-unified-plan-00) and [Plan
B](https://tools.ietf.org/html/draft-uberti-rtcweb-plan-00). Unified Plan and
Plan B were two competing IETF drafts for the negotiation and exchange of
multiple media sources (AKA MediaStreamTracks, or MSTs) between two WebRTC
endpoints. Unified Plan is being incorporated in the [JSEP
draft](https://tools.ietf.org/html/draft-ietf-rtcweb-jsep-09) and is on its way
to becoming an IETF standard while Plan B has expired in 2013 and nobody should
care about it anymore, right? Wrong!

Plan B lives on in the Chrome and its derivatives, like Chromium and Opera.
There's actually an issue in the Chromium bug tracker to add support for
[Unified Plan in
Chromium](https://code.google.com/p/chromium/issues/detail?id=465349), but
that'll take some time. Firefox, on the other hand, has, [as of
recently](https://hacks.mozilla.org/2015/03/webrtc-in-firefox-38-multistream-and-renegotiation/),
implemented Unified Plan.

Developers that want to support both Firefox and Chrome have to deal with this
situation and implement some kind of _interoperability_ layer between Chrome and
it derivatives and Firefox.

The most substantial difference between Unified Plan and Plan B is how they
represent media stream tracks. Unified Plan extends the standard way of
encoding this information in SDP which is to have each RTP flow (i.e., SSRC)
appear on its own m-line. So, each media stream track is represented by its own
unique m-line.  This is a strict one-to-one mapping; a single media stream
track cannot be spread across several m-lines, nor may a single m-line
represent multiple media stream tracks.

Plan B takes a different approach, and creates a hierarchy within SDP; a m=
line defines an "envelope", specifying codec and transport parameters, and
a=ssrc lines are used to describe individual media sources within that
envelope. So, typically, a Plan B SDP has three channels, one for audio, one
for video and one for the data.

### Installation

Install locally from npm:

```bash
$ npm install @blueromans/sdp-translator
```
or yarn

```bash
$ yarn add @blueromans/sdp-translator
```
## Implementation

This module gives a general solution to the problem of SDP interoperability
described in the previous section and deals with it at the lowest level. The idea
is to build a PeerConnection adapter that will feed the right SDP to the browser,
i.e. Unified Plan to Firefox and Plan B to Chrome and that would give a Plan B SDP
to the application.

### The SDP interoperability layer

sdp-interop is a reusable npm module that offers the two simple methods:

* `toUnifiedPlan(sdp)` that takes an SDP string and transforms it into a
  Unified Plan SDP.
* `toPlanB(sdp)` that, not surprisingly, takes an SDP string and transforms it
  to Plan B SDP.

The PeerConnection adapter wraps the `setLocalDescription()`,
`setRemoteDescription()` methods and the success callbacks of the
`createAnswer()` and `createOffer()` methods. If the browser is Chrome, the
adapter does nothing. If, on the other hand, the browser is Firefox the
PeerConnection adapter...

* calls the `toUnifiedPlan()` method of the sdp-interop module prior to calling
  the `setLocalDescription()` or the `setRemoteDescription()` methods, thus
  converting the Plan B SDP from the application to a Unified Plan SDP that
  Firefox can understand.
* calls the `toPlanB()` method prior to calling the `createAnswer()` or the
  `createOffer()` success callback, thus converting the Unified Plan SDP from
  Firefox to a Plan B SDP that the application can understand.

Here's a sample PeerConnection adapter:

```javascript
function PeerConnectionAdapter(ice_config, constraints) {
    var RTCPeerConnection = navigator.mozGetUserMedia
      ? mozRTCPeerConnection : webkitRTCPeerConnection;
    this.peerconnection = new RTCPeerConnection(ice_config, constraints);
    this.interop = new require('sdp-interop').Interop();
}

PeerConnectionAdapter.prototype.setLocalDescription
  = function (description, successCallback, failureCallback) {
    // if we're running on FF, transform to Unified Plan first.
    if (navigator.mozGetUserMedia)
        description = this.interop.toUnifiedPlan(description);

    var self = this;
    this.peerconnection.setLocalDescription(description,
        function () { successCallback(); },
        function (err) { failureCallback(err); }
    );
};

PeerConnectionAdapter.prototype.setRemoteDescription
  = function (description, successCallback, failureCallback) {
    // if we're running on FF, transform to Unified Plan first.
    if (navigator.mozGetUserMedia)
        description = this.interop.toUnifiedPlan(description);

    var self = this;
    this.peerconnection.setRemoteDescription(description,
        function () { successCallback(); },
        function (err) { failureCallback(err); }
    );
};

PeerConnectionAdapter.prototype.createAnswer
  = function (successCallback, failureCallback, constraints) {
    var self = this;
    this.peerconnection.createAnswer(
        function (answer) {
            if (navigator.mozGetUserMedia)
                answer = self.interop.toPlanB(answer);
            successCallback(answer);
        },
        function(err) {
            failureCallback(err);
        },
        constraints
    );
};

PeerConnectionAdapter.prototype.createOffer
  = function (successCallback, failureCallback, constraints) {
    var self = this;
    this.peerconnection.createOffer(
        function (offer) {
            if (navigator.mozGetUserMedia)
                offer = self.interop.toPlanB(offer);
            successCallback(offer);
        },
        function(err) {
            failureCallback(err);
        },
        constraints
    );
};
```