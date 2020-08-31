---
title: "Impl Sha256"
date: 2020-08-28T08:09:13-04:00
draft: true
---

How does SHA256 work?

What is a hash?
Given a messages M of m bits generate a data N of n bits where n < m. 
Must be collision resistant


What is cryptographic hash?
h = H(x) 
1. Large input domain -> small fixed output => compression
2. Well distributed P(H(x)-i) = 1/N

Three additional properties:
1. Pre-image resistant - given H(x), hard to find x.
2. Weak collision resistance - hard to find any x such that H(x) = h
3. Strong collision resistance - hard to find any pair such that H(x) = H(y)

How is hashing different from encryption?

What is a message schedule?
