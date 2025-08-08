---
layout: post
title:  "Spring AI with Perplexity and Ollama"
date:   2025-08-01 06:30:10 +02
categories: java spring_ai perplexity_ai ollama pgvector
published: true
---
This document birefly describe how a RAG setup is achived using `PgVector`, `Ollama` and `Perplexity` cloud APIs.
PgVector is an extesnin of PostgresSql data base where we can store embedding of our data which we can be use
to provide context for a RAG. Unlike Gemini, Peplexity does not have an embeding model(atleast ofr now) that we can use. Therefore, we can use a locally running `ollama' to create the vector embdedding of any given text. Once, we have a vector data for our data, we can use that as the context for any given query and pass along to the Perplexity cloud API.

I will divide the entire setup into 3 different parts so that we can easily understand resposnsiobilty of individual module.
You can find the entire code base and ready to run setup in the github [dishsuggester](https://github.com/deleSerna/ai-ex/tree/main/java/springAI/dishsuggester) sample.

As a first step, make a `PgVector` db up and running using docker. A `compose.yml` file is provided in the above github sample.  Go to the `dishsuggester` directory and start the db  using the `docker-compose up` command. If evrrything is working as expected then you should see something along the following lines in the log.

```
pgvector-doggydb  | 2025-06-30 21:09:54.274 UTC [1] LOG:  database system is ready to accept connections
```

You can also connect to the DB using psql cli to make sure everything is ok.
```psql -U YOUR_DOGGY_DB_USER   -h localhost doggybagdb```

Once the db is up and running we can start the `ollama` local instance via docker `docker compose -f ollama-compose.yml`.
If everything is OK, then check it's status in the browser `http://localhost:11434` and it should display
`Ollama is running`. Once the `ollama`is running then we should pull some `ollama` LLM so that we can use that to create the vector embedding. In our case, we can pull `nomic-embed-text`.
```
docker exec -it ollama  ollama pull nomic-embed-text
```

Once the model is pulled then we can check whether it's working by calculating vector embedding of some arbritrary text. We can use `curl` command for that purpose.
```
curl localhost:11434/api/embed -d "{\"model\":\"nomic-embed-text\",\"input\":\"what is the capital of france ?\"}"

```

Now we have a vector db and embedding model to generate the vector embedding is ready, we can now ingest our documents using the a springboot application. A similar write up to integrate springboot and Gemini vertetx api can be find [here](https://deleserna.github.io/java/spring_ai/vertex/ai/vaadin/2025/05/29/spring_ai_vertex.html).
Please modify the `build.gradle` and `application.yml` accordingly. Ingesting the given sample texts in the `resources/breakfast` directory is provided in the `IngestionPipeline` class. To control the `ingestion` and `deletion` of ingested documents, you can use some end points as provided in the `ApiRequestController`.

Once documents are injected, we can do `similaritySearch` to retrieve relevant documents from the embedded db and then use that as a context in the prompt.  At this point you can use any LLM provider to get the required information based on the given retrieved context. In our sample, we have used `Perpelxity` as the LLM provider but it is easily replacable.
I have subscribed to `Perplexity pro` account, hence used that. Get the `API` token as mentioned [here](https://www.perplexity.ai/help-center/en/articles/10352995-api-settings) and provide that in the `application.yml`. It is best to access sensitive info via environment variable rather than pasting directly in the project as once committed, it will be available in the git history.The implementation detail can be found at `ApiRequestController.chat(..)`.
