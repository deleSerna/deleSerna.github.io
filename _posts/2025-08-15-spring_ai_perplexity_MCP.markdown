---
layout: post
title:  "MCP server that connects Perpelxity chat model using Spring AI"
date:   2025-08-15 06:30:10 +02
categories: java spring_ai perplexity_ai mcp mcp_server mcp_inspector
published: false
---
This document briefly describe how a MCP server that can connect to the Perplexity chat model can be created using Spring boot and how to use mcp inspector as [MCP client](https://modelcontextprotocol.io/legacy/tools/inspector) to interact with them.

For this sample, we are only building a simple MCP server. Hence, there is not much complex logic on the server side.
In the `build.gradle`,
 - Add `org.springframework.ai:spring-ai-starter-mcp-server` for the MCP server related libraries.
 - Add `org.springframework.ai:spring-ai-starter-model-openai` for the Perplexity (OpenAI) chat model related libraries.
In the `application.properties`,
 - Enable STDIO transport by  disabling rest endpoints `spring.main.web-application-type=none`
 - Disable `logging`, otherwise logged messages will seen as output. STDIO protocol typically expects all input and output data to be in JSON format. If Spring Boot app writes logs or banners to stdout, those will mix with your JSON output and cause JSON parsing error. Logging can be disabled by the configuration `logging.pattern.console=` and spring boot ASCII art banner can be disabled by `spring.main.banner-mode=off`.
 - Add all the Perplexity related configuration like API key, model, url etc.

In the source code,
  - Expose all the tools that the server will provided. Those should be annotate with `@Tool` and should be groupped into respective service class.
  - Once the service classes are defined (EchoToolService and PerplexityToolService), register those as ToolCallback in the SpringBootApplication class. Some sources mentioned that you do not need to add it as a ToolCallback, if you annotate them with @Tool but for some reason, it did not work and I did not really explore further in that direction.
  - Perplexity is using the OpenAI library interfaces in the Spring boot project and ideally `OpenAIChatModel` that can connect clent can be automatically injected with the configured properties. But for some reason, it was not happening due to some kind of ambiguity and hence I explcitly OpenAiChatModel  that can connect to Perplexity is manually created.

Once all these are done,  you can  build the project using `./gradlew clean build` and the jar should be present at `mcp-server_stdio/build/libs/mcp-server_stdio-0.0.1-SNAPSHOT.jar` if build is successful.

Now the server part is done, we can connect to it using a MCP client. You can use MCP clients like Claude desktop etc.
But here I have used the [MCP insepctor](https://modelcontextprotocol.io/legacy/tools/inspector) as MCP client.


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


