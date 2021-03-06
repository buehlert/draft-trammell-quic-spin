---
title: The Addition of a Spin Bit to the QUIC Transport Protocol
abbrev: Spin Bit
docname: draft-trammell-quic-spin-latest
date:
category: info

ipr: trust200902
workgroup: QUIC
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
  -
    ins: B. Trammell
    role: editor
    name: Brian Trammell
    org: ETH Zurich
    email: ietf@trammell.ch
  -
    ins: P. De Vaere
    name: Piet De Vaere
    org: ETH Zurich
    email: piet@devae.re
  -
    ins: R. Even
    name: Roni Even
    org: Huawei
    email: roni.even@huawei.com
  -
    ins: G. Fioccola
    name: Giuseppe Fioccola
    org: Telecom Italia
    email: giuseppe.fioccola@telecomitalia.it
  -
    ins: T. Fossati
    name: Thomas Fossati
    org: Nokia
    email: thomas.fossati@nokia.com
  -
    ins: M. Ihlar
    name: Marcus Ihlar
    org: Ericsson
    email: marcus.ihlar@ericsson.com
  -
    ins: A. Morton
    name: Al Morton
    org: AT&T Labs
    email: acmorton@att.com
  -
    ins: E. Stephan
    name: Emile Stephan
    org: Orange
    email: emile.stephan@orange.com

normative:

informative:
  TRILAT:
    title: On the Suitability of RTT Measurements for Geolocation (https://github.com/britram/trilateration/blob/paper-rev-1/paper.ipynb)
    author:
      -
        ins: B. Trammell
    date: 2017-08-30
  TOKYO-PING:
    title: From Paris to Tokyo - On the Suitability of ping to Measure Latency (ACM IMC 2014)
    author:
      -
        ins: C. Pelsser
      -
        ins: L. Cittadini
      -
        ins: S. Vissicchio
      -
        ins: R. Bush
    date: 2014-10-23
  CARRA-RTT:
    title: Passive Online RTT Estimation for Flow-Aware Routers Using One-Way Traffic (NETWORKING 2010, LNCS 6091, pp. 109–121)
    author:
      -
        ins: D. Carra
      -
        ins: K. Avrachenkov
      -
        ins: S. Alouf
      -
        ins: A. Blanc
      -
        ins: P. Nain
      -
        ins: G. Post
    date: 2010
  CONUS:
    title: Comparison of Backbone Node RTT and Great Circle Distances (https://github.com/acmacm/FIXME-TBD)
    author:
      -
        ins: A. Morton
    date: 2017-09-01
  NOSPIN:
    title: Description of a tool chain to evaluate Unidirectional Passive RTT measurement (and results) (https://github.com/acmacm/PassiveRTT)
    author:
      -
        ins: A. Morton
    date: 2017-10-05
  SPINBIT-REPORT:
    title: Latency Spinbit Implementation Experience
    author:
      -
        ins: P. De Vaere
    date: 2017-11-28
  MINQ:
    title: MINQ, a simple Go implementation of QUIC (https://github.com/ekr/minq)
    author:
      -
        ins: E. Rescorla
    date: 2017-11-28
  MOKUMOKUREN:
    title: Mokumokuren, a lightweight flow meter using gopacket (https://github.com/britram/mokumokuren)
    author:
      -
        ins: B. Trammell
    date: 2017-11-12
  IMC-CONGESTION:
    title: Challenges in Inferring Internet Interdomain Congestion (in Proc. ACM IMC 2014)
    author:
      -
        ins: M. Luckie
      -
        ins: A. Dhamdhere
      -
        ins: D. Clark
      -
        ins: B. Huffaker
      -
        ins: k. claffy
    date: 2014-11
  IMC-TCPSIG:
    title: TCP Congeston Signatures (ACM IMC 2017)
    author:
      -
        ins: S. Sundaresan
      -
        ins: A. Dhamdhere
      -
        ins: M. Allman
      -
        ins: k claffy
  CACM-TCP:
    title: Passively Measuring TCP Round-Trip Times (in Communications of the ACM)
    author:
      -
        ins: S. Strowes
    date: 2013-10
  TMA-QOF:
    title: Inline Data Integrity Signals for Passive Measurement (in Proc. TMA 2014)
    author:
      -
        ins: B. Trammell
      -
        ins: D. Gugelmann
      -
        ins: N. Brownlee
    date: 2014-04
  WWMM-BLOAT:
    title: Impact of TCP Congestion Control on Bufferbloat in Cellular Networks (in Proc. IEEE WoWMoM 2013)
    author:
      -
        ins: S. Alfredsson
      -
        ins: G. Giudice
      -
        ins: J. Garcia
      -
        ins: A. Brunstrom
      -
        ins: L. Cicco
      -
        ins: S. Mascolo
    date: 2013-06

--- abstract

This document summarizes work to date on the addition of a "spin bit",
intended for explicit measurability of end-to-end RTT on QUIC flows. It
proposes a detailed mechanism for the spin bit, describes how to use it to
measure end-to-end latency, discusses corner cases and workarounds therefor in
the measurement, describes experimental evaluation of the mechanism done to
date, and examines the utility and privacy implications of the spin bit. As
the overhead and risk associated with the spin bit are negligible, and the
utility of a passive RTT measurement signal at higher resolution than once per
flow is clear, this document advocates for the addition of the spin bit to the
protocol.

--- middle

# Introduction

The QUIC transport protocol {{?QUIC-TRANS=I-D.ietf-quic-transport}} is a
UDP-encapsulated protocol integrated with Transport Layer Security (TLS)
{{?TLS=I-D.ietf-tls-tls13}} to encrypt most of its protocol internals, beyond
those handshake packets needed to establish or resume a TLS session, and
information required to reassemble QUIC streams (the packet number) and to
route QUIC packets to the correct machine in a load-balancing situation (the
connection ID). In other words, in contrast to TCP, QUIC's wire image (see
{{?WIRE-IMAGE=I-D.trammell-wire-image}}) exposes much less information about
transport protocol state than TCP's wire image. Specifically, the fact that
sequence and acknowledgement numbers and timestamps cannot be seen by
on-path observes in QUIC as they can be in the TCP means that passive TCP loss
and latency measurement techniques that rely on this information (e.g.
{{CACM-TCP}}, {{TMA-QOF}}) cannot be easily ported to work with QUIC.

This document proposes a solution to this problem by adding a "latency spin
bit" to the QUIC short header. This bit is designed solely for explicit
passive measurability of the protocol. It provides one RTT sample per RTT to
passive observers of QUIC traffic. It describes the mechanism, how it can be
added to QUIC, and how it can be used by passive measurement facilities to
generate RTT samples. It explores potential corner cases and shortcomings of
the mechanism and how they can be worked around. It summarizes experimental
results to date with an implementation of the spin bit built atop a recent
QUIC implementation. It additionally describes use cases for passive RTT
measurement at the resolution provided by the spin bit. It further reviews
findings on privacy risk researched by the QUIC RTT Design Team, which was
tasked by the IETF QUIC Working Group to determine the risk/utility tradeoff
for the spin bit.

The spin bit has low overhead, presents negligible privacy risk, and has clear
utility in providing passive RTT measurability of QUIC that is far superior to
QUIC's measurability without the spin bit, and equivalent to or better than
TCP passive measurability.

## About This Document

This document is maintained in the GitHub repository
https://github.com/britram/draft-trammell-quic-spin, and the editor's copy is
available online at https://britram.github.io/draft-trammell-quic-spin.
Current open issues on the document can be seen at
https://github.com/britram/draft-trammell-quic-spin/issues. Comments and
suggestions on this document can be made by filing an issue there, or by
contacting the editor.

# The Spin Bit Mechanism {#mechanism}

The latency spin bit enables latency monitoring from observation points on
the network path. The bit is set by the endpoints in the following way:

* The server sets the spin bit value to the value of the
  spin bit in the packet received from the client with
  the largest packet number.

* The client sets the spin bit value to the opposite
  of the value set in the packet received from the server with the
  largest packet number, or to 0 if no packet as been received yet.

If packets are delivered in order, this procedure will cause the spin bit
to change value in each direction once per round trip. Observation points can
estimate the network latency by observing these changes in the latency spin
bit, as described in {{usage}}.

## Proposed Short Header Format Including Spin Bit

Since it is possible to measure handshake RTT without a spin bit (see
{{handshake}}), it is sufficient to include the spin bit in the short
packet header. This proposal suggests to use the second most significant bit
(0x40) of the first octet in the short header for the spin bit.

~~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+
|0|C|K|S|Type(4)|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                     [Connection ID (64)]                      +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Packet Number (8/16/32)                ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Protected Payload (*)                   ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~
{: #fig-short-header title="Short Header Format including proposed Spin Bit"}

This will limit the number of available short packet types to 16. The short
packet types will be redefined to the following values:

| Type | Packet Number Size |
|:-----|:-------------------|
|  0xD | 4 octets           |
|  0xE | 2 octets           |
|  0xF | 1 octet            |
{: #short-packet-types title="Short Header Packet Types after Definition of Spin Bit"}

Note that this proposal changes the short header as defined in the editor's
copy of {{QUIC-TRANS}} at the time of writing; regardless of where and how
the spin bit is eventually defined, the key properties of the spin bit are (1)
it's a single bit, (2) it spins as defined above, and (3) it appears only in
the short header; i.e. after version negotiation and connection establishment
are completed.

# Using the Spin Bit for Passive RTT Measurement {#usage}

When a QUIC flow is sending at full rate (i.e., neither application nor flow
control limited), the latency spin bit in each direction changes value once
per round-trip time (RTT). An on-path observer can observe the time difference
between edges in the spin bit signal to measure one sample of end-to-end RTT.
Note that this measurement, as with passive RTT measurement for TCP, includes
any transport protocol delay (e.g., delayed sending of acknowledgements)
and/or application layer delay (e.g., waiting for a request to complete). It
therefore provides devices on path a good instantaneous estimate of the RTT as
experienced by the application. A simple linear smoothing or moving minimum
filter can be applied to the stream of RTT information to get a more stable
estimate.

We note that the Latency Spin Bit, and the measurements that can be done with
it, can be seen as an end-to-end extension of a special case of the alternate
marking method described in {{?ALT-MARK=I-D.ietf-ippm-alt-mark}}.

## Limitations and Workarounds {#limitations}

Application-limited and flow-control-limited senders can have application and
transport layer delay, respectively, that are much greater than network RTT.
Therefore, the spin bit provides network latency information only when the
sender is neither application nor flow control limited. When the sender is
application-limited by periodic application traffic, where that period is
longer than the RTT, measuring the spin bit provides information about the
application period, not the RTT. Simple heuristics based on the observed data
rate per flow or changes in the RTT series can be used to reject bad RTT
samples due to application or flow control limitation.

Since the spin bit logic at each endpoint considers only samples on packets
that advance the largest packet number seen, signal generation itself is
resistent to reordering. However, reordering can cause problems at an observer
by causing spurious edge detection and therefore low RTT estimates. This can
be probabilistically mitigated by the observer tracking the low-order bits of
the packet number, and rejecting edges that appear out-of-order.

## Experimental Evaluation

We have evaluated the effectiveness of the spin bit in an emulated network
environment. The spin bit was added to a fork of {{MINQ}}, using the mechanism
described in {{mechanism}}, but with the spin bit appearing in a measurement
byte added to the header for passive measurability experiments. Spin bit
measurement support was added to {{MOKUMOKUREN}}. Full results of these
ongoing experiments are available online in {{SPINBIT-REPORT}}, but we
summarize our findings here.

First, we confirm that the spin bit works as advertised: it provides one
useful RTT sample per RTT to any passive observer of the flow. This sample
tracks each sender's local instantaneous estimate of RTT as well as the
expected RTT (i.e., defined by the emulation) fairly well. One surprising
implication of this is that the spin bit provides _more_ information than is
available by local estimation to an endpoint which is mostly receiving data
frames and sending mainly ACKs, and as such can also be useful in purely
endpoint-local observations of the RTT evolution during the flow. The spin bit
also works correctly under moderate to heavy packet loss and jitter.

Second, we confirm that the spin bit can be easily implemented without
requiring deep integration into a QUIC implementation. Indeed, it could be
implemented completely independently, as a shim, aside from the requirement
that the spin bit value be integrity-protected along with the rest of the QUIC
header.

Third, we performed experiments focused on the intermittent-sender problem
described in {{limitations}}. We confirm that the spinbit does not provide
useful RTT samples after the handshake when packets are only sent
intermittently. Simple heuristics can be used to recognize this situation,
however, and to reject these RTT samples. We also find that a simple
sender-side heuristic can be used to determine whether a sample will be
useful. If a sender sends a packet more than a specified delay (e.g. 1ms)
after the last packet received by the client, it knows that any latency spin
observation of that packet will be invalid. If a second "spin valid" bit were
available, the sender could then mark that packet "spin invalid". Our
experiments show that this simple heuristic and spin validity bit are
succesful in marking all packets whose RTT samples should be rejected.

Fourth, we performed experiments focused on the reordering problem described
in {{limitations}}. We find that while reordering can cause spurious samples
at a naive observer, two simple approaches can be used to reject spurious RTT
samples due to reordering. First, a two-bit spin signal that always advances
in a single direction (e.g. 00 -> 01 -> 10 -> 11) successfully rejects all
reordered samples, including under amounts of reordering that render the
transport itself mostly useless. However, adding a bit is not necessary:
having the observer keep the least significant bits of the packet number, and
rejecting samples from packets that do not advance by one, as suggested in
{{limitations}}, is essentially as successful as a two-bit spin signal in
mitigating the effects of reordering on RTT measurement.

Fifth, we performed parallel active measurements using ping, as described in
{{just-ping-it}}. In our emulated network, the ICMP packets and the QUIC
packets traverse the same links with the same treatment, and share queues at
each link, which mitigates most of the issues with ping. We find that while
ping works as expected in measuring end-to-end RTT, it does not track the
sender's estimate of RTT, and as such does not measure the RTT experienced by
the application layer as well as the spin bit does.

In summary, our experiments show that the spin bit is suitable for purpose,
can be implemented with minimal disruption, and that most of the problems
identified with it in specific corner cases can be easily mitigated. See
{{SPINBIT-REPORT}} for more.

# Use Cases for Passive RTT Measurement

This section describes use cases for passive RTT measurement. Most of these
are currently achieved with TCP, i.e., the matching of packets based on
sequence and acknowledgment numbers, or timestamps and timestamp echoes, in
order to generate upstream and downstream RTT samples which can be added to
get end-to-end RTT. These use cases could be achieved with QUIC by replacing
sequence/acknowledgement and timestamp analysis with spin bit analysis, as
described in {{usage}}.

In any case, the measurement methodology follows one of a few basic variants:

- The RTT evolution of a flow or a set of flows can be compared to baseline or
  expected RTT measurements for flows with the same characterisitcs in order
  to detect or localize latency issues in a specific network.

- The RTT evolution of a single flow can also be examined in detail to
  diagnose performance issues with that flow.

- The spin bit can be used to generate a large number of samples of RTT for a
  flow aggregate (e.g., all flows between two given networks) without regard
  to temporal evolution of the RTT, in order to examine the distribution of
  RTTs for a group of flows that should have similar RTT (e.g., because they
  should share the same path(s)).

## Interdomain Troubleshooting

Network access providers are often the first point of contact by their
customers when network problems impact the performance of bandwidth-intensive
and latency-sensitive applications such as video, regardless of whether the
root cause lies within the access provider's network, the service provider's
network, on the Internet paths between them, or within the customer's own
network.

Many residential networks use WiFi (802.11) on the last segment, and WiFi
signal strength degradation manifests in high first-hop delay, due to the fact
that the MAC layer will retransmit packets lost at that layer. Measuring the RTT
between endpoints on the customer network and parts of the service provider's
own infrastructure (which have predictable delay characteristics) can be used
to isolate this cause of performance problems.

Comparing the evolution of passively-measured RTTs between a customer network
and selected other networks on the Internet to short- and medium-term baseline
measurements can similarly be used to isolate high latency to specific
networks or network segments. For example, if the RTTs of all flows to a given
content provider increase at the same time, the problem likely exists between
the access network and the content provider, or in the content provider's
network itself. On the other hand, if the RTTs of all flows passing through
the same access provider infrastructure change together, then the change is
likely attributable to that infrastructure.

These measurements are particularly useful for traffic which is latency
sensitive, such as interactive video applications. However, since high latency
is often correlated with other network-layer issues such as chronic
interconnect congestion {{IMC-CONGESTION}}, it is useful for general
troubleshooting of network layer issues in an interdomain setting.

In this case, multiple RTT samples per flow are useful less for observing
intraflow behavior, and more for generating sufficient samples for a given
aggregate to make a high-quality measurement.

## Bufferbloat Mitigation in Cellular Networks

Cellular networks consist of multiple Radio Access Networks (RAN) where
mobile devices are attached to base stations. It is common that base stations
from different vendors and different generations are deployed in the same
cellular network.

Due to the dynamic nature of RANs, base stations have typically been
provisioned with large buffers to maximize throughput despite rapid changes in
capacity. As a side effect, buffer bloat has become a common issue in such
networks {{WWMM-BLOAT}}.

An effective way of mitigating buffer bloat without sacrificing too much
throughput is to deploy Active Queue Management (AQM) in bottleneck routers and
base stations. However, due to the variation in deployed base-stations it is
not always possible to enable AQM at the bottlenecks, without massive
infrastructure investments.

An alternative approach is to deploy AQM as a network function in a more
centralized location than the traditional bottleneck nodes. Such an AQM
monitors the RTT progression of flows and drops or marks packets when the
measured latency is indicative of congestion. Such a function also has the
possibility to detect misbehaving flows and reduce the negative impact they have
on the network.

## Quality of Experience (QoE) monitoring for media streams

\[EDITOR'S NOTE: see https://github.com/britram/draft-trammell-quic-spin/issues/8]

## Third-Party Latency Verification

\[EDITOR'S NOTE: see https://github.com/britram/draft-trammell-quic-spin/issues/6]

## Internet Measurement Research

As a large, distributed, engineered system with no centralized control, the
Internet has emergent properties of interest to the research community not
just for purely scientific curiosity, but also to provide applicable guidance
to Internet engineering, Internet protocol design and development, network
operations, and policy development. Latency measurements in particular are
both an active area of research as well as an important tool for certain
measurement studies (see, e.g. {{IMC-TCPSIG}}, from the most recent Internet
Measurement Conference). While much of this work is currently done with active
measurements, the ability to generate latency samples passively or using a
hybrid measurement approach (i.e., through passive observation of
purpose-generated active measurement traffic; see {{?RFC7799}}) can
drastically increase the efficiency and scalability of these studies. A
latency spin bit would make these techniques applicable to QUIC, as well.

# Alternate RTT Measurement Approaches for Diagnosing QUIC flows {#other-bad-ideas}

There are three broad alternatives to explicit signaling for passive RTT
measurement for measuring the RTT experienced by QUIC flows.

## Handshake RTT measurement {#handshake}

The first of these is handshake RTT measurement. As described in
{{?QUIC-MGT=I-D.ietf-quic-manageability}}, the packets of the QUIC handshake
are distinguishable on the wire in such a way that they can be used for one
RTT measurement sample per flow: the delay between the client initial and the
server cleartext packet can be used to measure "upstream" RTT (between the
observer and the server), and the delay between the server cleartext packet
and the next client cleartext packet can be used to measure "downstream" RTT
(between the client and the observer). When RTT measurements are used in large
aggregates (all flows traversing a large link, for example), a methodology
based on handshake RTT could be used to generate sufficient samples for some
purposes without the spin bit.

However, this methodology would rely on the assumption that the difference
between handshake RTT and nominal in-flow RTT is negligible. Specifically, (1)
any additional delay required to compute any cryptographic parameters must be
negligible with respect to network RTT; (2) any additional delay required to
establish state along the path must be negligible with respect to network RTT;
and (3) network treatment of initial packets in a flow must identical to that
of later packets in the flow. When these assumptions cannot be shown to hold,
spin-bit based RTT measurement is preferable to handshake RTT measurement,
even for applications for which handshake RTT measurement would otherwise be
suitable.

## Parallel active measurement {#just-ping-it}

The second alternative is parallel active measurement: using ICMP Echo Request
and Reply {{?RFC0792}} {{?RFC4433}}, a dedicated measurement protocol like
TWAMP {{?RFC5357}}, or a separate diagnostic QUIC flow to measure RTT.
Regardless of protocol, the active measurement must be initiated by a client
on the same network as the client of the QUIC flow(s) of interest, or a
network close by in the Internet topology, toward the server. Note that there
is no guarantee that ICMP flows will receive the same network treatment as the
flows under study, both due to differential treatment of ICMP traffic and due
to ECMP routing (see e.g. {{TOKYO-PING}}). TWAMP and QUIC diagnostic flows,
though both use UDP, have similar issues regarding ECMP. However, in
situations where the entity doing the measurement can guarantee that the
active measurement traffic will traverse the subpaths of interest (e.g.
residential access network measurement under a network architecture and
business model where the network operator owns the CPE), active measurement
can be used to generate RTT samples at the cost of at least two non-productive
packets sent though the network per sample.

## Frequency Analysis {#frequency-analysis}

The third alternative, proposed during the QUIC RTT design team process,
relies on the inter-packet spacing to convey information about the RTT, and
would therefore allow measurements confined to a single direction of
transmission, as described in {{CARRA-RTT}}.

We evaluated the applicability of this work to passive RTT measurement in
QUIC, and found it wanting. We assebled a toolchain, as described in
{{NOSPIN}}, that allowed evaluation of a critical aspect of the {{CARRA-RTT}}
method: extraction of inter-packet times of real packet streams and the
analysis of frequencies present in the packet stream using the Lomb-Scargle
Periodogram. Several streams were evaluated, as summarized below:

* It seems that Carra et al. {{CARRA-RTT}} took the noisy and low-confidence
  results of a statistical process (no RTT-related frquency has been detected
  even after using very low alpha confidence) and added hueristics with
  sliding-window averaging to infer the fundamental frequency and RTT present
  in a unidirectional stream.

* There appear to be several limitations on the streams that are applicable.
  Streams with long RTT (~50ms) are more likely to be suitable (having a
  better match between packet rate and relatively low frequencies to detect).

* None of the TCP streams analysed (to date) posess a sufficient packet rate
such that the measured fundamental frequency or the multiples of the fundamental
are actually within the detectable range.

* "Ideal" interarrival time streams were simulated with uniform sampling and
  period. The Lomb-Scargle Periodogram is surprisingly unable to detect the
  fundamental frequency at 100 Hz from the constant 10 ms packet spacing.

* It is not clear if IETF QUIC protocol stream will possess the same
  inter-packet arrival time features as TCP streams. Also, Carra et al. note
  that their process may not work if the TCP stream encounters a bottleneck,
  which would be the essential case for network troubleshooting. Mobile
  networks with time-slot service disciplines would likely cause similar
  issues as a bottleneck, by imposing the time-slot interval on the spacing of
  many packets.

* The Carra et al. {{CARRA-RTT}} calculation of minimum and maximum frquecies
  that can be detected may not be applicable when the inter-arrival times are
  (both) the signal being detected and govern the non-uniform sampling
  frequency.

# Privacy and Security Considerations

The privacy considerations for the latency spin bit are essentially the same
as those for passive RTT measurement in general.

A concern was raised during the discussion of this feature within the QUIC
working group and the QUIC RTT Design Team that high-resolution RTT
information might be usable for geolocation. However, an evaluation based on
RTT samples taken over 13,780 paths in the Internet from RIPE Atlas anchoring
measurements {{TRILAT}} shows that the magnitude and uncertainty of RTT data
render the resolution of geolocation information that can be derived from
Internet RTT is limited to national- or continental-scale; i.e., less
resolution than is generally available from free, open IP geolocation
databases.

One reason for the inaccuracy of geolocation from network RTT is that Internet
backbone transmission facilities do not follow the great-circle path between
major nodes. Instead, major geographic features and the efficiency of
connecting adjacent major cities influence the facility routing. An evaluation
of ~3500 measurements on a mesh of 25 backbone nodes in the continental United
States shows that 85% had RTT to great-circle error of 3ms or more, making
location within US State boundaries ambigous {{CONUS}}.

Therefore, in the general case, when an endpoint's IP address is known, RTT
information provides negligible additional information.

RTT information may be used to infer the occupancy of queues along a path;
indeed, this is part of its utility for performance measurement and
diagnostics. When a link on given path has excessive buffering (on the order
of hundreds of milliseconds or more; a situation colloquially referred to as
"bufferbloat"), such that the difference in delay between an empty queue and a
full queue dwarfs normal variance and RTT along the path, RTT variance during
the lifetime of a flow can be used to infer the presence of traffic on the
bottleneck link. In practice, however, this is not a concern for passive
measurement of congestion-controlled traffic, since any observer in a
situation to observe RTT passively need not infer the presence of the traffic,
as it can observe it directly.

In addition, since RTT information contains application as well as network
delay, patterns in RTT variance from minimum, and therefore application delay,
can be used to infer or fingerprint application-layer behavior. However, as
with the case above, this is not a concern with passive measurement, since the
packet size and interarrival time sequence, which is also directly observable,
carries more information than RTT variance sequence.

We therefore conclude that the high-resolution, per-flow exposure of RTT for
passive measurement as provided by the spin bit poses negligible marginal risk
to privacy.

As shown in {{mechanism}}, the spin bit can be implemented separately from the
rest of the mechanisms of the QUIC transport protocol, as it requires no
access to any state other than that observable in the QUIC packet header
itself. We recommend that implementations take advantage of this property, to
reduce the risk that a errors in the implementation could leak private
transport protocol state through the spin bit.

Since the spin bit is disconnected from transport mechanics, a QUIC endpoint
implementing the spin bit that has a model of the actual network RTT and a
target RTT to expose can "lie" about its spin bit transitions, even without
coordination with and the collusion of the other endpoint. This is not the
case with TCP, which requires coordination and collusion to expose false
information via its sequence and acknowledgment numbers and its timestamp
option. When passive measurement is used for purposes where one endpoint might
gain a material advantage by representing a false RTT, e.g. SLA verification
or enforcement of telecommunications regulations, this situation raises a
question about the trustworthiness of spin bit RTT measurements.

This issue must be appreciated by users of spin bit information, but
mitigation is simple, as QUIC implementations designed to lie about RTT
through spin bit modification are subject to dynamic analysis along paths with
known RTTs. We consider the ease of verification of lying in situations where
this would be prohibited by regulation or contract, combined with the
consequences of violation of said regulation or contract, to be a sufficient
incentive in the general case not to do it.

# Acknowledgments

Many thanks to Christian Huitema, who originally proposed the spin bit as pull
request 609 on {{QUIC-TRANS}}. Thanks to the QUIC RTT Design Team for
discussions leading especially to the measurement limitations and privacy and
security considerations sections.

This work is partially supported by the European Commission under Horizon 2020
grant agreement no. 688421 Measurement and Architecture for a Middleboxed
Internet (MAMI), and by the Swiss State Secretariat for Education, Research,
and Innovation under contract no. 15.0268. This support does not imply
endorsement.
