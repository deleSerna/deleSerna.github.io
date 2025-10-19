---
layout: post
title:  "MCP Orchestrator"
date:   2025-10-17 23:30:10 +02
categories: java spring_ai spring_boot mcp ollama perplexity_ai agentic_patterns
published: true
---
This is a follow up of my [previous article](https://deleserna.github.io/java/spring_ai/perplexity_ai/mcp/mcp_server/mcp_inspector/2025/08/15/spring_ai_perplexity_MCP.html). Here, I demonstrate how we could set up multiple MCP client-servers and how an agentic pattern can be used to integrate them.

The overall architecture of the setup is as follows. We have 2 MCP servers each exposeing different tools but connecting to the same db to retrieve different data. Each MCP server is connected to an MCP client via http. The server will announce its exposed tool to the client and here we use ollama to select the tools provided by the server. In our example, MCP server exposes only one tool, therefore here the client won't have much difficulty in choosing the right tool. But we can easily expand the MCP server to expose more tools.

Each MCP client is dedicatedly connected to only one MCP server. Therefore, we use an orchestrator/router that can select one among the MCP Clients using a perplexity-ai model based on user input.

All the code can be found at my [github repo](https://github.com/deleSerna/ai-ex/tree/main/java/springAI/mcporchestrator).

Orchestrator is a modified version from the [spring ai example](https://github.com/spring-projects/spring-ai-examples/tree/main/agentic-patterns/routing-workflow).
MCP client and server example is adapted based on this [github project](https://github.com/kuldeepsingh99/mcp-server-with-spring-ai/blob/main/README.md).

# MCP server

- Make the potsgres db up and running.
   - Go the [docker folder](https://github.com/deleSerna/ai-ex/tree/main/java/springAI/mcporchestrator/mcp-client-server/docker) and `docker-compose up -d`
   - Connect to the db `psql -U myuser -h localhost mydatabase`.
   - Enter the password which is present in the docker file.
   - Create the table by `\i path-to-table.sql`
   - Insert data in the generated table by `\i path-to-insert-data.sql`
   - Verify that data is generated as expected by `select * from public.seller_account;`

- Make the ollama db up and running.
   - docker-compose -f ollama-compose.yml up -d
   - Pull the  llama 3.1 by `docker exec -it ollama_mcp ollama pull llama3.1`

- Go to each [mcpserver](https://github.com/deleSerna/ai-ex/tree/main/java/springAI/mcporchestrator/mcp-client-server/mcp-server) folder and run ` ./gradlew bootRun` and make sure that server is started as expected.

# MCP client
Go the [mcp client folder](https://github.com/deleSerna/ai-ex/tree/main/java/springAI/mcporchestrator/mcp-client-server/mcp-client) and run ` ./gradlew bootRun` and make sure that the client is started as expected. Verify that MCP client to server connection is ok by sending a request to the MCP client. 
  - `curl -X GET -H "Content-Type: text/plain" -G --data-urlencode "q=Marc" http://localhost:8040/name`

# MCP Orchestrator

Fill the PerplexityAPI credentials in the  [application.properties](https://github.com/deleSerna/ai-ex/blob/main/java/springAI/mcporchestrator/routing-workflow/src/main/resources/application.properties#L5) and start the server by `./mvnw spring-boot:run`. Verify that the orchestrator is working as expected by sending different requests to make sure that you are getting responses from both MCP clients
 - To verify it's connected to MCP client 1, sent a request something like below.
  `curl -X GET -H "Content-Type: text/plain" -G --data-urlencode "message=find out a person with name David" http://localhost:8080/route/agent`
The tool found one account matching the name "David". The details of this account are:
```
ID: 1
Name: David
Owner: Amazon
Type: STD
Status: 1
```
 - To verify it's connected to MCP client 2, send a request something like below.
` curl -X GET -H "Content-Type: text/plain" -G --data-urlencode "message=find out a company  with name flipkart" http://localhost:8080/route/agent`
Based on the tool's output, there is one seller account owned by Flipkart with the following details:
```
*   ID: 2
*   Name: Marc
*   Owner: flipkart
*   Type: STD (Standard)
*   Status: 1                                                            
```

Based on the example, we can see that the orchestrator is automatically picking up the right client based on the user input and thus we are connected to different MCP servers for different input.
We could easily adapt these sample approaches for different MCP servers.


**References**
1. [MCP orchestrator](https://github.com/spring-projects/spring-ai-examples/tree/main/agentic-patterns/routing-workflow)
2. [MCP server - client](https://github.com/kuldeepsingh99/mcp-server-with-spring-ai/blob/main/README.md)