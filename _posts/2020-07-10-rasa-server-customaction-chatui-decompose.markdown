---
layout: post
title:  "Deploy rasa server, custom action server and chatbot ui"
date:   2020-07-10 23:40:36 +0530
categories: rasa chatbot docker-decompose
---

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
** rasa server docker file
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

** rasa action server docker file

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
** chat bot html file 
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
** chatbot ui docker file

```
FROM nginx

COPY index.html /usr/share/nginx/html

```

** endpoints.yml
```

action_endpoint:
  url: "http://action_server:5055/webhook"
```

** credentials.yml
```
socketio:
  user_message_evt: user_uttered
  bot_message_evt: bot_uttered
  session_persistence: true
```

[ to be updated]

References
1.https://rasa.com/docs/rasa/user-guide/building-assistants/
2. https://rasa.com/docs/rasa/user-guide/how-to-deploy/#building-an-action-server-image
3. https://github.com/botfront/rasa-webchat
