# MODBUS Protocol Wiki

A comprehensive knowledge base for the MODBUS protocol, for developers building MODBUS services or applications.  All information on this site is generated directly from the MODBUS specifications, using AI software.   See _COPYRIGHT NOTICE_ below for a statement about the copyright status.
## Overview

MODBUS is an application-layer messaging protocol (OSI Layer 7) providing client/server communication between devices. Originally developed by Modicon in 1979 for programmable logic controllers, it remains the de facto standard for industrial device communication.

This repository transforms official MODBUS specification documents into an easily navigable wiki format, making it simple for software developers to:
- Understand MODBUS protocol fundamentals
- Learn transport variants (TCP, RTU, ASCII, Security)
- Access function code specifications
- Implement correct data models
- Follow best practices for industrial communication

## How to use the MODBUS Wiki

The first stage of using this wiki is to simply browse the content on this site.

* [`wiki/index.md`](./wiki/index.md) -- View the main page for the Wiki
* [`wiki/summaries`](./wiki/summaries/README.md) -- Summaries of the MODBUS specification documents
* [`wiki/concepts`](./wiki/concepts/README.md) -- Concepts in the MODBUS specifications
* [`wiki/answers`](./wiki/answers/README.md) -- Answering common questions about using MODBUS

The content in the `wiki` directory is designed to be easy to browse.

The wiki may be downloaded to your computer and used by tools like Claude Code or Open Code for AI chat purposes.

```shell
$ git clone https://github.com/clean-energy-tools/modbus-wiki.git
$ cd modbus-wiki
# This example uses Claude Code, it should work with any
# similar tool like Open Code.
$ claude
           Claude Code v2.1.114
 ▐▛███▜▌   Sonnet 4.6 · Claude Pro
▝▜█████▛▘  ~/Projects/clean-energy-tools/modbus-wiki
  ▘▘ ▝▝    Welcome to Opus 4.7 xhigh! · /effort to tune speed vs. intelligence
  
❯ explain how to use MODBUS in a program

● Let me check the project context first.
  Searched for 1 pattern, read 2 files (ctrl+o to expand)
● Here's how to use MODBUS in a program, covering the key areas:                                                         
  1. Choose Your Transport                                                         
  - MODBUS TCP — connect over TCP/IP to port 502; simplest for modern systems
  - MODBUS RTU — serial (RS-485/RS-232); common for legacy industrial devices
  - MODBUS ASCII — serial with ASCII framing; less common

...
```

The answer to the above question actually goes on to give a long and elaborate answer.

NOTE: The scope of information used in generating answers is almost entirely the content of the specification documents.

The process used by the AI tool to generate answers is described in the _Question answering_ section of AGENTS.md.  You can experiment with changes to this process, if you like.

If the AI agent gives a particularly good answer, you may add that answer to your version of the wiki by writing the following request:

```
Create an answer document from the information you just gave
```

The AI system will create a suitable file in `wiki/answers`, and make sure the answer document is cross-linked as appropriate to other files in the wiki.

You may also contribute content to the wiki.  You do this by opening a pull request with the GitHub repository.
## Repository Structure

```
modbus-ai-wiki/
├── README.md                          # This file
├── AGENTS.md                          # Ingestion workflow documentation
├── raw/                               # Source documents (immutable)
│   └── MODBUS/
│       ├── MODBUS.md                    # Comprehensive AI-optimized reference
│       ├── modbusprotocolspecification.md    # Official application protocol spec
│       ├── modbusoverserial.md              # Serial line specification
│       ├── messagingimplementationguide.md     # TCP/IP implementation guide
│       └── modbussecurityprotocol.md       # Security protocol spec
└── wiki/                              # Maintained wiki pages
    ├── index.md                      # Table of contents
    ├── log.md                        # Change log
    ├── summaries/                    # Document summaries
    │   └── MODBUS/
    ├── answers/                      # Answers to questions about MODBUS
    └── concepts/                     # Concept pages
```

The files in `raw/MODBUS` are a mechanical conversion of the official PDFs into Markdown.  The result isn't meant for reading by humans.  If you want to read the original specification, go directly to the MODBUS website https://www.modbus.org/specifications 

## Resources

### Official MODBUS Resources

- [MODBUS Organization](https://www.modbus.org/) - Official MODBUS website
- [MODBUS Specifications](https://www.modbus.org/modbus-specifications) - Official specifications
- [MODBUS FAQ](https://www.modbus.org/faq) - Frequently asked questions

### Related Standards

- IEC 61158: Industrial communication networks - Fieldbus specifications
- IEEE 802.3: Ethernet standard
- TIA/EIA-485: RS-485 electrical standard

### Community

- Stack Overflow: MODBUS protocol questions
- Industrial automation forums
- GitHub: MODBUS implementations and libraries
## Version History

See [wiki/log.md](wiki/log.md) for detailed change history.

**Current Version:** 1.0.0 (2026-04-18)
- Initial ingestion of 5 MODBUS specification documents
- 19 wiki pages created (5 summaries, 12 concepts, 2 navigation)
- ~30,000 words of technical content

## Support

For questions or issues:
1. Check relevant wiki pages first
2. Refer to official MODBUS specifications
3. Search community forums
4. Consult device vendor documentation

## Acknowledgments

This wiki is based on specifications maintained by the [MODBUS Organization](https://www.modbus.org/). Content is extracted and organized for software developer convenience.

## COPYRIGHT NOTICE

This wiki summarizes public MODBUS specifications. The MODBUS protocol is maintained by the MODBUS Organization. Refer to official specifications for licensing and usage rights.

The MODBUS specifications are owned by, copyright by, Modbus Organization, Inc.  The PDFs were downloaded from https://www.modbus.org/, converted to Markdown, then processed using the instructions in [AGENTS.md](./AGENTS.md).  

I am not a Lawyer, but I think the status of the contents of this wiki are:

- The `raw/` directory is directly covered by the MODBUS Organization copyright
- The `wiki/` directory contains information derived from the official specifications.  These may or may not be covered by the MODBUS Organization copyright.  The information in this wiki is offered not with the intent of infringing on the rights of the MODBUS Orgnization or Schneider, but of providing value to those interested in understanding the MODBUS specifications.


