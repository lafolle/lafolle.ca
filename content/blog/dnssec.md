---
title: "Did you send it?"
date: 2025-04-05T01:20:00-08:00
slug: "dnssec"
description: ""
keywords: []
draft: false
tags: []
math: false
toc: false
---


Is DNS secure without DNSSEC??? No.  All queries made by clients to recursive resolver, authoritative name servers, TLD name servers or root
name servers are un-encrypted.
Apart from that DNS is based on UDP protocol which is connectionless in nature such that if i have a DNS server running on my
(public) machine anyone who knows the IP and port on which the server is listening for traffic, can easily send request to it.

For example,  lets say rrdns sends a query to `ns1.google.com` to fetch A records for _google.com_ zone from a socket which
is binded to `localhost:30000`.  In case of happy path one of the machines corresponding to `ns1.google.com` wil reply back
with list of A records for _google.com_.  But an evil person can easily hijack the request and send back response
with IP address corresponding to some evil-website because they easily would be able to read queries as they fly around the internet un-encrypted.

The onus is on the client to verify the response it received for the query it sent.

Welcome to DNSSEC.

## Digital Signatures

The basic problem that we intend to solve is, how to verify the origin of message ie, if Alice sends a message to Bob then how
can Bob be sure that the message he received is truely from Alice?  This is plain old problem in cryptography that is solved by Digital Signatures which goes like this:
1. Alice generates a pair of keys: `public_key_alice` and `private_key_alice`.
2. Alice gives the `public_key_alice` to Bob.
3. Alice signs the message using `private_key_alice`.
4. Alice sends the generated signature in 3 along with the actual message `M` to Bob.
5. Bob receives the message and verifies the signature using `public_key_alice`.

If malicious Eve comes in between and changes dns responss, the verification at Bob's end would fail.  Would it be possible for Eve to change both
data and signature? Yes,  but then she would also need to know Alice's private key to be able to generate signature which can be assertively verified by Bob.  The private key really needs to be well protected or the whole system would break.  And if it ever does then a new key-pair needs to be generated immidiately.

## Digital Signatures in DNS

In DNS, each zone maintains its own RSA key-pair and signs the RRSet (set of resource records for given type and owner) of domains it hosts with its private key.  When client makes a query to the DNS server, the server returns the answers (RRSets) and also their corresponding signatures.  The client uses zone's public key to verify the RRSets and the signature in response.

The zone's key-pair is known as zone-signing key-pair,  denoted by ZSK.  The signature of the RRSet is stored in resource record whose type is RRSIG while the ZSK's public is published by zone in DNSKEY record.


# Reference
1. [DNSSEC Complexities and Considerations](https://www.cloudflare.com/dns/dnssec/dnssec-complexities-and-considerations/)
2. [How dnssec works](https://www.cloudflare.com/dns/dnssec/how-dnssec-works/)


_Note: originally written on 2020-08-19, publishing it today_































