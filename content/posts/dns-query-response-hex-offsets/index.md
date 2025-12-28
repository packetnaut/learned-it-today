---
title: "Deeper Dive: DNS Query and Response with Wireshark and tcpdump with HEX Offsets"
date: 2024-05-26
tags: ["dns", "tcpdump", "wireshark", "packet-analysis"]
draft: false
---

Hello everyone, this is my first post. I am doing the Protocol Deep Dive: DNS course on Pluralsight by Betty DuBois. It was very well explained — the course duration is 2h 21mins but she gives so much relevant information I take almost 1–2 hours to do a 20 min video.

In my attempts to sharpen my tcpdump skills, I decided to look at the packet examples both in Wireshark and tcpdump to decipher the tcpdump gibberish I saw and also dive deeper into the DNS packets.

## The DNS Query

Let's dive right in, the first packet we will look at is a DNS query.

In Wireshark it looks like this. Notice in order to filter down the packets to a relevant packet you are interested in you can use the filter: `dns.qry.name contains "XXX"`. Here we can see the transaction ID of our query:

<!-- IMAGE: Wireshark DNS query showing transaction ID -->

To do this in tcpdump, this is our basic command as well as a few additional flags I added. In green we see the decimal equivalent of the transaction ID:

<!-- IMAGE: tcpdump output showing transaction ID -->

```bash
tcpdump -r PCAP -s0 -Snnvvl | grep 'sales'
```

Let's break this down:

- `-r PCAP`: Read packets from a file rather than capturing live packets from a network interface. Replace `PCAP` with the actual file name of the packet capture file you want to analyze.
- `-s0`: Sets the snapshot length to 0, meaning `tcpdump` will capture the entire packet. By default, `tcpdump` only captures the first 96 bytes of each packet, but setting it to 0 ensures that no part of the packet is truncated.
- `-S`: Print absolute sequence numbers in TCP packets instead of relative sequence numbers. Helpful for detailed analysis.
- `-nn`: Do not resolve hostnames and service names. Prints numeric IP addresses and port numbers, which speeds up the output and avoids DNS lookups.
- `-v`: Increases the verbosity level, providing more detailed information about each packet.
- `-l`: Makes the output line-buffered, useful when piping the output to another command (like `grep` in this case).
- `grep 'sales'`: Pipes the output through `grep`, which searches for and displays only the lines that contain the string 'sales'.

## DNS Flags

Moving on to the flags, here we can see it is a standard query:

- If **response** is set to 0, it is a query
- If **Opcode** is also set to 0, it is a standard query
- If **truncated** is set, it means the packet was too large to send via UDP, so the packet was sent in TCP with multiple packets (however they may not be received in the same order it was sent)
- **Non-authenticated data** means the client wants the server to check with the authoritative server first to ensure the data is good, and the server will respond from cache

In order to convert `0x0100` to get the associated value for each flag field:

```
|q| OP   |a|t|r|r|z|a|c|       |
|r| code |a|c|d|a|Z|d|d| Rcode |
```

`0x0100`: Similar to calculating TCP flags, you need to convert each digit to binary:
- 0 → 0000
- 1 → 0001
- 0 → 0000
- 0 → 0000

Combined: `0000 0001 0000 0000` — same as seen in Wireshark.

## Question and Resource Record Counts

We have only 1 question, along with no Answer RRs, Authority RRs, or Additional RRs. These will be set in the reply. Here you can see it represented in HEX:

- `00 01` for Questions
- `00 00` for Answer RRs
- `00 00` for Authority RRs
- `00 00` for Additional RRs

## Query Labels

Moving on to the query, we can see the query length is 15 characters and it has 4 labels. Each label is the actual word seen in the query, for example:

- label 1: sales
- label 2: gt
- label 3: gm
- label 4: com

When looking at the HEX prior to each label, you can see the length of the actual label.

<!-- IMAGE: Query labels with hex length prefixes -->

Let's look into the additional characteristics such as type and class of the query.

## The DNS Response

Let's dive into the response. As you can see, the transaction ID matches.

In Wireshark it appears like this:

<!-- IMAGE: Wireshark DNS response -->

So far it has been observed that the `*` attached to the query ID indicates an authoritative answer was observed in the DNS query response, meaning the responding server is an authority for the domain (NOT 100% sure).

Moving forward after the query, we see `2/0/0` in tcpdump. This means:
- Answer RRs = 2
- Authority RRs = 0
- Additional RRs = 0

Since the setup is load balanced, we receive responses from both. Depending on the load balance mechanism used, two different servers can respond to the same host queries. In green you can see the query along with the labels, the type and the class as shown above for the query.

<!-- IMAGE: tcpdump response showing 2/0/0 -->

## DNS Compression: The 0xc00c Pointer

Following this, we see `0xc00c`. If you check to the left, it actually shows this value for the query name. Since the repetition of domain names is common in DNS responses, a compression mechanism is used to reduce the size of the packet.

- The value `0xc0` indicates that this is a pointer
- `0x0c` is the offset of where the query name starts

If there are different answers provided, the first query name will be at `0x0c`, however additional query name pointers will vary depending on where the query name was first defined in the packet.

**IMPORTANT:** This offset counter starts at the beginning of the DNS section in the packet, not at the start of the UDP packet. If we count, we add 12:

- Transaction ID: 2 bytes
- Flags: 2 bytes + 2 bytes (from above) = 4 bytes
- Questions: 2 bytes + 4 bytes (from above) = 6 bytes
- Answer RRs: 2 bytes + 6 bytes (from above) = 8 bytes
- Authority RRs: 2 bytes + 8 bytes (from above) = 10 bytes
- Additional RRs: 2 bytes + 10 bytes (from above) = 12 bytes

Then we will be at the start of the query name: **12 bytes = 0x0c**

## Multiple Answers

The same applies for the next answer. Since it is the exact query name as the previous answer, the pointer is assigned (`0xc00c`) the same offset as before.

The only difference between the two answers is the IP addresses:
- First answer: `0x0a0c0308` = 10.12.3.8
- Second answer: `0x0a0c0312` = 10.12.3.18

<!-- IMAGE: Multiple answers with different IPs -->

To further explain the pointer: if I have multiple answers with different query names because of DNS lookups, the query names are already defined in the previous answers and are being referenced with the pointer and the offset position.

<!-- IMAGE: Multiple answers with different query names -->

**Important to note:** The offset counter is cumulative, meaning it is counting offset positions from the last pointer to be able to get to the next query. So `0xc02e` is 46 positions from the beginning of the query, and 34 positions after the last pointer.

Same applies here: the pointer `0xc058` is 88 positions from the beginning of the query, and 42 positions after the last pointer.

<!-- IMAGE: Offset calculation example -->

## References

- [tcpdump man page](https://www.tcpdump.org/manpages/tcpdump.1.html)
