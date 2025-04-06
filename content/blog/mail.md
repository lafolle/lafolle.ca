---
title: "You've got mail"
date: 2025-03-28T12:55:42-07:00
draft: true
---

## How to fetch emails?

Two protocols are used to download emails:

IMAP (Internet Message Access Protocol) is for fetching emails from mail server. Mail stays in mail server once the emails are fetched. This is good for cases where multiple mail clients intend to fetch emails. For example, Gmail app and web interface can be considered two different mail clients, conceptually, and both can use IMAP to download emails.

POP3 (Post Office Protocol 3) is also for fetching emails from mail server. Unlike IMAP, mails are deleted from mail server once fetched. Only a single mail client can access mails using this protocol.

## How to send email?

SMTP (Simple Mail Transfer Protocol) is for sending mails to mail server.

# Email format

Here is a sample email conversation between sender@gmail.com and receiver@zoho.com. Note, the sender is using Gmail while the receiver is using Zoho. For simplicity, all conversations are in plain text.

> sender@gmail.com > receiver@zoho.com: aksfjlaksdjf ladfj aldfj alskdfj lasjalf jl

> receiver@zoho.com > sender@gmail.com: Hello there, we couldn't understand your email.

> sender@gmail.com > receiver@zoho.com: oh sorry, can you send me my subscription information please?

How are these emails connected?

First message is the original message. Second msg is reply to first and third msg is reply to second.
