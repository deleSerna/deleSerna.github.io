---
layout: post
title:  "Deploy rasa server, custom action server and chatbot ui"
date:   2020-07-10 23:40:36 +0530
categories: rasa chatbot docker-decompose
---
I have recently developed a custom rasa chat bot by following [rasa tutorial][rasa-1]. I went bit further than explained in the tutorial and created custom action which depend on a **duckling server** and connected a chat ui to talk to the bot. When I tried to deploy the all these together using **docker-compose**, I could not find a resource which directly explain how to create custom images of my chat bot and deploy it. Therefore I thought of sharing docker files which I used. I am not going through how to create a custom chat bot because that's nicely explained [here][rasa-1].

[rasa-1]:https://rasa.com/docs/rasa/user-guide/building-assistants/
Once you are ready with the chat bot, we should deploy rasa core server in one docker container and action server in another. 

We will first create the action server's image by following [link][rasa-2]. We copy **action** file into a separate action folder and dependency will be added in the requirements file.
[rasa-2]:https://rasa.com/docs/rasa/user-guide/how-to-deploy/#building-an-action-server-image
**rasa action server docker file**

```
# Extend the official Rasa SDK image
FROM rasa/rasa-sdk:1.6.1

# Use subdirectory as working directory
WORKDIR /app

# Copy any additional custom requirements, if necessary (uncomment next line)
COPY actions/requirements-actions.txt ./

# Change back to root user to install dependencies
USER root

# Install extra requirements for actions code, if necessary (uncomment next line)
RUN pip install -r "requirements-actions.txt"

# Copy actions folder to working directory
COPY ./actions /app/actions

# By best practices, don't run the code with root user
USER 1001

# Start the action server
CMD ["start", "--actions", "actions", "--debug"]
```
Now we will create the docker image of rasa server file (core + nlu). We have to update **endpoints.yml** with service name (the one which we add in docker-compose) of action server and we have to enable **socketio** in **credentails.yml** so that chat ui can connect to it. If you are using some other ui like Facebook or slack then we we have to enable respective lines in **credentials.yml**. We have to copy following files domain.yml, config.yml, credentials.yml, endpoints.yml and data etc. to the docker image and then we will train the server to create the model.

**endpoints.yml**
```

action_endpoint:
  url: "http://action_server:5055/webhook"
```

**credentials.yml**
```
socketio:
  user_message_evt: user_uttered
  bot_message_evt: bot_uttered
  session_persistence: true
```

**rasa server docker file**
```
FROM rasa/rasa:1.6.1-spacy-en

WORKDIR /app
COPY . /app
COPY ./data /app/data

USER root
RUN  rasa train

VOLUME /app
VOLUME /app/data
VOLUME /app/models
USER 1001
CMD [ "rasa","shell" ]

```
To connect to the rasa server, I used ui provided by https://github.com/botfront/rasa-webchat. I just wrapped the script provided by them in a html file and deployed it in a nginx web server.

**chat bot html file** 
```
<body>
  <div id="webchat"></div>
  <script src="https://cdn.jsdelivr.net/npm/rasa-webchat/lib/index.min.js"></script>
  <script> // script taken from https://github.com/botfront/rasa-webchat
    WebChat.default.init({
      selector: "#webchat",
      initPayload: "/get_started",
      customData: {"language": "en"}, // arbitrary custom data. Stay minimal as this will be added to the socket
      socketUrl: "http://localhost:5005",
      socketPath: "/socket.io/",
      title: "Title",
      subtitle: "Subtitle",
      params: {"storage": "session"} // can be set to "local"  or "session". details in storage section.
    })
  </script>
</body>

```
**chatbot ui docker file**

```
FROM nginx

COPY index.html /usr/share/nginx/html

```

Finally, we will deploy everything together using a docker-compose as follows. If deploy went fine then we can access the chat ui using  http://localhost/ and talk to the bot.

**docker-compose.yaml**
```
version: '3.0'
services:
  rasa_server:
    image: myrasaserver:v1
    ports:
      - 5005:5005
    depends_on:
      - duckling
      - action_server
    command:
      - run
      - -m
      - /app/models
      - --cors
      - "*"
      - --enable-api
      - --log-file
      - out.log
      - -p
      - '5005'
  action_server:
    image: myactionserver:v1
    ports:
      - "5055:5055"
    command:
      - start
      - --actions
      - actions
  ui_nginx:
    image: botui:v1
    ports:
      - "80:80"
  duckling:
    image: rasa/duckling
    ports:
      - "8000:8000"

```

**References**

 1. https://rasa.com/docs/rasa/user-guide/building-assistants/
 2. https://rasa.com/docs/rasa/user-guide/how-to-deploy/#building-an-action-server-image
 3. https://github.com/botfront/rasa-webchat
