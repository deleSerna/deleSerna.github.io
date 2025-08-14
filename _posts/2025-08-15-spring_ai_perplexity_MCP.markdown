---
layout: post
title:  "MCP server that connects Perpelxity chat model using Spring AI"
date:   2025-08-15 06:30:10 +02
categories: java spring_ai perplexity_ai mcp mcp_server mcp_inspector
published: false
---
---
layout: post
title:  "Spring AI with Perplexity and Ollama"
date:   2025-08-01 06:30:10 +02
categories: java spring_ai perplexity_ai ollama pgvector
published: true
---
This document briefly describe how a MCP server that can connect to the Perplexity chat model can be created using Spring boot and how to use mcp inspector as [MCP client](https://modelcontextprotocol.io/legacy/tools/inspector) to interact with them.

MCP Server side notes
 - Deependencies in the build.gradle
 - In application.properties, disable rest endpoints ( enable stdio) , disable logging and add perplexity configuration.
 - Add all the services
 - Create OpenAiChatModel explicitly as constructor injection was cnot not working due to ambiguity. Hence used Primary bean approach.



MCP client
 - Install npm  ( for mac use bre install npm)
 - Start npx @modelcontextprotocol/inspector . Dont provide any the jars as arument here as it did not work for me.
 - Open the UI, choose STDIO as transport
   - Command 'java' . Please be careful to avoid extrspaces  at both end otherwise, it wont connect to the jar
   -  args -jar replace-with-full-path/mcp-server_stdio/build/libs/mcp-server_stdio-0.0.1-SNAPSHOT.jar` 
   - Press the connect button, if connection is successful then go to tools tab and list out the tools and use it.

 If connection is unsucessful then it's best to stand alone run the   `java -jar replace-with-full-path/mcp-server_stdio/build/libs/mcp-server_stdio-0.0.1-SNAPSHOT.jar` and add enough debugging statement to make sure that it is running as expected.
 If the above jar is working as expected and still you cuould connect then its' time to check whether inspector has some issue.
 The easiest would be to create some java script that just spawn java and see if that works and continue form there on.
 Mcp inspector [ github  page] (https://github.com/modelcontextprotocol/inspector/issues) is active one, you could get help from there.


