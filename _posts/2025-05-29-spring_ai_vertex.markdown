---
layout: post
title:  "Spring AI with Vertex 101"
date:   2025-05-29 06:30:10 +02
categories: java spring_ai vertex ai 
published: false
---

Follow this [guide](https://spring.io/guides/gs/spring-boot#scratch) from spring and create a sample application. I have also added `spring-boot-starter-web` and `spring-ai-starter-model-vertex-ai-gemini` etc. as the dependency. Then, download the project
and open it in your favorite IDE. I have chosen gradle instead of maven. The dependency block in the build.gradle will look like something below.

```groovy
	implementation 'org.springframework.boot:spring-boot-starter-web'
	implementation("org.springframework.ai:spring-ai-starter-model-vertex-ai-gemini:1.0.0")
```
Then create a sample controller that uses a VertexAiGeminiChatModel. You can use a sample from [here][Spring AI sample] or [here](https://github.com/deleSerna/ai-ex/tree/main/java/springAI).

Then to connect to Vertex Gemini API, following properties are need to add to `resources/application.properties`.
```
spring.ai.vertex.ai.gemini.project-id=<fill-your-project-id>
spring.ai.vertex.ai.gemini.location=<your-location>
spring.ai.vertex.ai.gemini.chat.options.model=gemini-2.0-flash
spring.ai.vertex.ai.gemini.chat.options.temperature=0.5
```
Then as mentioned [here][Google-Cloud], create a project in the google cloud and follow the below steps.
 - In the Google Cloud console, create or select a Google Cloud project.
- Ensure that Billing is enabled for your project. Generative AI models usually require a billed account. At the time of writing, Google is providing a 300$ free credit to use it.
- In the console, enable the Vertex AI API.

Get the project id and your nearest google cloud location from the Google Cloud console and fill it in your `application.properties`.

We should also configure our system to login to Google cloud as we need to login to use the Vertex API. Please follow the below steps for that.

- In your terminal, install the [gcloud CLI](https://cloud.google.com/sdk/docs/install?utm_campaign=CDR_0xff63493c_default_b417970735&utm_medium=external&utm_source=blog), which is essential for managing resources and setting up authentication for local development where you run your application.
Set up your application default credentials by running the following commands: 
```

# initialize gcloud 
gcloud init
# set the Project ID you have configured
gcloud config set project <PROJECT_ID>
# authenticate with your user account
gcloud auth application-default login <GMAIL ID>
```
Now run the application using IDE or by running `./gradlew bootRun`.

Now we can go to the browser or cli and goto the url to check whether it's working or not.
```
Type the following URL in the browser and it should output something like {"generation":"Good morning! How can I help you today?\n"}
http://localhost:8080/ai/generate?message=goodmorning
```
Voila. You are ready to go.
**References**

[Google-Cloud]: https://cloud.google.com/blog/topics/developers-practitioners/google-cloud-and-spring-ai-10
[Spring AI sample]: https://code.likeagirl.io/spring-boot-ai-chat-app-with-deepseek-openai-gemini-d860bb8fe5cb
