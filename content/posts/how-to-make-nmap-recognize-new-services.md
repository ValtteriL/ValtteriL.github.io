---
title: "How to Make Nmap Recognize New Services"
description: "Step-by-step instructions for extending nmap service detection capabilities"
date: 2024-03-03T16:36:06+02:00
draft: false
tags: [
    "nmap",
    "opc ua",
    "guide"
]
categories: [
    "Guide",
]
---

## How to Make Nmap Recognize New Services

Nmap has been my favorite hacking tool for years. Its accuracy is unchallenged and it boasts hundreds of scripts that make it vital in every pentest engagement.

Lately, I've been working more on the ICS space, developing a [OPC UA vulnerability scanner](https://opalopc.com). To my dismay, I noticed that Nmap does not recognize OPC UA services. This makes black box security testing of this dominating ICS protocol tricky, as OPC UA server vendors are known to use non-standard ports extensively.

Having read the [Nmap book](https://nmap.org/book/), I knew it wouldn't be too hard to teach it how to detect new services. Having used Nmap for a long time it was also time to pay back. Therefore I decided to contribute the protocol detection to the Nmap codebase and write a short tutorial to show how you can do the same for other unrecognized protocols. What follows is that tutorial.

## Getting a copy of the codebase

1. Create a public fork of the [Nmap repository](https://github.com/nmap/nmap) (requires a GitHub account).
2. Clone your fork: `git clone <your fork>`
3. Enter the clone `cd nmap`
4. Verify your target service is **not** recognized with the latest probes file(requires nmap):

```bash
# The idea is to avoid having to compile Nmap
# by making changes to the nmap-service-probes file
# and test-running it with your system Nmap

mkdir my-dir
cp nmap-service-probes my-dir/
sudo nmap -sV --version-all -Pn --datadir my-dir -p <PORT> <TARGET>

# example result where service is not recognized:
#
# PORT      STATE SERVICE    VERSION
# 53530/tcp open  unknown
```

My starting point was as follows:

{{< figure src="/images/nmap-not-recognizing-opc-ua.png" alt="Nmap output without OPC UA service detection capability" caption="Nmap not recognizing the service" >}}

Nmap detects that the port is open, but the service is reported as `unknown`.

## Learning about the protocol

We need to know the protocol to teach Nmap to detect it.

In OPC UA's case, there was a convenient [deep dive](https://claroty.com/team82/research/opc-ua-deep-dive-part-3-exploring-the-opc-ua-protocol) into the protocol by Claroty. This is where I learned that the OPC UA session starts with a HELLO handshake. This led me to the OPC Foundation's [specification](https://reference.opcfoundation.org/Core/Part6/v105/docs/7.1.2) on Message structure, which laid out the blueprint for the message and its possible responses.

By teaching Nmap to send the HELLO message (service probe), it will elicit responses from OPC UA servers. Then we can teach it how the responses will look like (service match), so it can determine the protocol.

The information in the specification is enough to develop the service probe and matches for Nmap. The HELLO message requires an EndpointUrl, which the receiving server will check and return an error if it's too long or unrecognized. Nmap can thus a message with an invalid URL and detect the OPC UA server if it responds with an error message.

## Adding service probe

1. Write a script that sends the packet you have in mind. In my case:

```python
#!/usr/bin/python

from struct import *
import socket

def str_to_bytes(s):
    return bytes(s, 'utf-8')

# message header
message_type = str_to_bytes("HEL")
reserved = str_to_bytes("F")

# message body
protocol_version = 0
receive_buffer_size = 8192
send_buffer_size = 8192
max_message_size = 0
max_chunk_count = 0
endpoint_url = str_to_bytes("opc.tcp://nmap.org:1337")

# 5 * 4 for the integers
# len(endpoint_url) for the string
# 8 for the header
# 4 for endpoint_url length
message_size = 5 * 4 + len(endpoint_url) + 8 + 4

hellomessage = pack(f"3ssIIIIIII{len(endpoint_url)}s",
                    message_type,
                    reserved,
                    message_size,
                    protocol_version,
                    receive_buffer_size,
                    send_buffer_size,
                    max_message_size,
                    max_chunk_count,
                    len(endpoint_url),
                    endpoint_url)

HOST = "172.16.1.8"
PORT = 53530

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.connect((HOST, PORT))
    s.sendall(hellomessage)
    print(f"Sent {hellomessage!r}")
    data = s.recv(1024)

print(f"Received {data!r}")
```

2. Verify you get the intended response:

```text
Sent b'HELF7\x00\x00\x00\x00\x00\x00\x00\x00 \x00\x00\x00 \x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x17\x00\x00\x00opc.tcp://nmap.org:1337'
Received b'ERRF\x83\x00\x00\x00\x00\x00\x83\x80s\x00\x00\x00Bad_TcpEndpointUrlInvalid (code=0x80830000, description="The server does not recognize the QueryString specified.")'
```

3. Record the message exchange with Wireshark (save the recording!)
4. In Wireshark, right-click the packet with your message, select `follow TCP stream`, filter to show only sent messages, select to show data as C arrays, copy the C array contents

{{< figure src="/images/wireshark-follow-stream-show-as-c-arrays.png" alt="Wireshark following TCP flow and copying data as C array" caption="Captured HELLO message as C Array" >}}

5. Paste the copied array into a text editor, remove commas and spaces, and replace the leading `0` in hex with `\`. It is then your probestring:

```text
\x48\x45\x4c\x46\x37\x00\x00\x00\x00\x00\x00\x00\x00\x20\x00\x00\x00\x20\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x17\x00\x00\x00\x6f\x70\x63\x2e\x74\x63\x70\x3a\x2f\x2f\x6e\x6d\x61\x70\x2e\x6f\x72\x67\x3a\x31\x33\x33\x37
```

6. Open the copy of `nmap-service-probes` and add the probestring with rarity and ports according to the [instructions](https://nmap.org/book/vscan-fileformat.html). In my case:

```text
# OPC UA TCP HEL message
Probe TCP HelRequest q|\x48\x45\x4c\x46\x37\x00\x00\x00\x00\x00\x00\x00\x00\x20\x00\x00\x00\x20\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x17\x00\x00\x00\x6f\x70\x63\x2e\x74\x63\x70\x3a\x2f\x2f\x6e\x6d\x61\x70\x2e\x6f\x72\x67\x3a\x31\x33\x33\x37|
rarity 6
ports 49320,62541,4897,53530,48050,4885,4840,4855,26543
```

I chose the rarity based on gut feeling. The ports I picked from [here](https://claroty.com/team82/research/opc-ua-deep-dive-series-a-one-of-a-kind-opc-ua-exploit-framework).

## Adding service match

1. Review the same TCP stream you copied the probe from, filter to show only received messages, select to show data as C arrays, copy the C array contents
2. Paste the copied array into a text editor, remove commas and spaces, and replace the leading `0` in hex with `\`.
3. Add the match **below** the probe in `nmap-service-probes` according to the same instructions (section match Directive).

In my case, the response contains a standard header, but each protocol implementation is free to choose the message body and therefore I cannot use the response as-is. Instead, I chose to match only the 4 first characters of the response. In the error message, the response is `ERR` followed by reserved byte `F`.

The only other valid response to the probe is an acknowledgment (`ACK` followed by reserved byte `F`). A server may respond with it, thus I chose to add it as a second match string.

This resulted in the following service matches:

```text
# Possible OPC UA TCP responses to HEL message
match opc-ua-tcp m|^ACKF|
match opc-ua-tcp m|^ERRF|
```

## Testing

Having added the probe and matches, nmap should be able to detect the protocol now.

1. Scan the service again with the updated probe file: `sudo nmap -sV --version-all -Pn --datadir my-dir -p <PORT> <TARGET>`

My response was as follows:

{{< figure src="/images/nmap-recognizing-opc-ua.png" alt="Nmap output with new service detection capability" caption="Nmap recognizing the service" >}}

The service is now detected as `opc-ua-tcp`.

## Publishing the changes

To make this improvement available for all nmap users, we need to create a pull request.

1. Checkout to a new branch: `git checkout -b <your-descriptive-branch-name>`
2. Overwrite the old `nmap-service-probes` with the improved one: `cp my-dir/nmap-service-probes nmap-service-probes`
3. Commit the `nmap-service-probes` change: `git add nmap-service-probes && git commit`
4. Add Git remote: `git remote add upstream https://github.com/nmap/nmap`
5. Push the change: `git push --set-upstream origin <your-descriptive-branch-name>`
6. Click the link in the output of the previous command to be taken to Github to create the PR
7. Write an informative description and submit the PR. You can see mine [here](https://github.com/nmap/nmap/pull/2791)
8. Send a notification email to `dev@nmap.org` referencing the PR and including a short description of the functionality of the patch

Congrats on the contribution! Now we are waiting for possible improvement suggestions or merging to the master branch. It will be a while before your change is on distro package manager version of Nmap!
