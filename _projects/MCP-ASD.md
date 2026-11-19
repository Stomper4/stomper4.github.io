---
layout: page
title: MCP Attack Surface Detector
description: Contributing to an open source Burp extension.
img: assets/img/projects/MCPAttackSurfaceDiscovery_WebHero.webp
importance: 1
category: Tooling
related_publications: false
---

[MCP Attack Surface Detector](https://github.com/hoodoer/MCP-ASD) (MCP-ASD) is an awesome tool developed by Hoodoer. You can find his blog about it [here](https://trustedsec.com/blog/mcp-in-burp-suite-from-enumeration-to-targeted-exploitation).

MCP-ASD provides a virtual proxy capable of translating asynchronous transport protocols to burpsuite's expected stateless HTTP protocol. This allows web application testers to use familiar tools and methodologies (such as Burp Intruder) against MCP servers which implement asynchronous transports such as websocket or SSE.

In my contribution I added support for the Streamable HTTP transport. This transport replaced SSE in the [MCP Specification](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http) in 2025. Different versions of this spec require different headers and metadata fields.

{% include figure.liquid loading="eager" path="/assets/img/projects/MCP_ASD1.png" class="img-fluid rounded z-depth-1" caption="Streamable HTTP is now shown in the transport menu." %}

The Streamable HTTP transport negotiates the protocol version with the server and remembers the accepted version. This allows the tool to automatically add the required HTTP headers and/or JSON metadata fields for each request.

Support for the Streamable HTTP transport allows this tool to be used for training alongside the [OWASP Damn Vulnerable MCP Server (DVMCP)](https://vwad.owasp.org/app/damn-vulnerable-mcp-server-dvmcp/).

My GitHub PR: [here](https://github.com/hoodoer/MCP-ASD/pull/11). The project is awaiting merge as the official version is currently on a code freeze while Portswigger evaluates it for addition to the BApp store. 

<br>
*Thanks for Reading*