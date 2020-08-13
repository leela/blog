---
title: "Setup Python Dev Instance Using Docker Compose"
date: 2020-08-12T16:11:00+05:30
draft: false
---
Technology stack is keep on changing. Few years back I was working on python projects without even using virtual environments. things changed very quickly that now we run all our projects on Docker. 

<!--more-->

You may have questions like is docker required for development instances? Let us go through this together and see why and how docker is useful in development.

Let me demonstrate this with very simple example. I am going to create a simple project that generates arithmatic graphs from the given numbers. I have one fastAPI application that simply returns the numbers and another streamlit web application that takes the numbers and generate a simple graph.

Here is my project structure
``` text
arithmetic-graphs
  - numbers
    - main.py
    - requirements.txt
  - web
    - main.py
    - requirements.txt
```
- `arithmetic-graphs` is a project where all our applications reside.
- we have two applications named `numbers` and `web`.
- Application logic stays in `main.py` module in every application.
- Every application has `requirements.txt`, that has list of python packages required for the application.

## Create and run project without docker

### Create and activate virtual environment
It is a good idea to have isolated environment for a project so that it can work independently.

Go to project directory and run these commands
``` text
arithmetic-graphs $ python -m venv env

arithmetic-graphs $ ls
api env web

arithmetic-graphs $ source env/bin/activate
(env)  arithmetic-graphs (master) $
```
- Create and activate virtual environment for the project

### Create numbers application

Here is `numbers/main.py` looks like:
``` python
from fastapi import FastAPI
import pandas as pd
import random
import json

app = FastAPI()

@app.get("/numbers")
def numbers():
    """Generates an array of size 10.
    """
    df = pd.DataFrame(random.sample(range(100), 10), columns=['A'])
    return json.loads(df.to_json())
```
- It is a fastAPI application
- `app` is created by calling `FastAPI` class
- Added `/numbers` endpoint that returns json.

Let's add required python packages to `numbers/requirements.txt`
``` text
fastapi
uvicorn
pandas
```

Voila, our numbers application is ready to run. Make sure that virtual environment is activated before you run the applciation.

Let's run the application
``` text
(env) arithmetic-graphs $ cd numbers/
(env) numbers $ pip install -r requirements.txt
...

(env) arithmetic-graphs $ uvicorn main:app
...
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
....
```
- install all the  packages required for application in virtual environment
- run the application using uvicorn

Go to uvicorn running url, in my case it is `http://127.0.0.1:8000` and can see the app working.
Access the endpoint we have added using `http://127.0.0.1:8000/numbers` and you will see a json response.


### Create web application

Here is `web/main.py` looks like:
``` python
import os
import streamlit as st
import pandas as pd

df = pd.read_json(os.environ['NUMBERS_URL'])
df['A/2'] = df['A']/2
st.line_chart(df)
```
- Running streamlit application
- Getting the numbers url from environment variable called `NUMBERS_URL`
- Modify dataframe to add some more arithmatic opearaions
- draw a line chart on the generated dataframe

Let's add required python packages to `web/requirements.txt`
``` text
streamlit
pandas
```
Voila, our web application is ready to run. Make sure that virtual environment is activated before you run the applciation.

Let's run the application
``` text
(env) arithmetic-graphs $ cd web/
(env) web $ export NUMBERS_URL=http://127.0.0.1:8000/numbers
(env) web $ pip install -r requirements.txt
...
(env) web $ streamlit run main.py
...
Local URL: http://localhost:8501
...
```
- Make sure that you set env variable `NUMBERS_URL` to the numbers endpoint coming from numbers app
- Install all the  packages required for application in virtual environment
- run the application using streamlit command

When you go to the url that streamlit given (in my case http://localhost:8501), you can see graph representation of the numbers coming from numbers api.

### What we have seen so far
* We have created two dependent applications named `numbers` and `web`
* We have made the project up and working

### All is well, should we stop here?
There are few steps  like creating virtual environment, installing requiremens are one time jobs. but there are few steps those we need to do everytime when you run your dev instance.

In case of `numbers` application, you run `uvicorn` command and for `web` app you make sure that environment variable is set and run `streamlit` command.

real world projects are usually more complicated than our simple project. In real world projects, you may need caching, databases, storage and many more micro services etc. To run a dev instance, you should make sure that all these are correctly setup in your machine. Isn't it lot of work?

What if we get our project as a package where we don't need to worry about whether the service is up or not? databse is configured correctly or not. 

Yes, that will be really cool, When you onboard a person you handover a package and tell how to use the package, no infrastructure worries. no need of installing python, database, cashe server etc. that will be awesome! Docker can help us doing that.

## Docker

Please make sure that you have docker installed. You can get docker from [here](https://docs.docker.com/get-docker/).

Let's add a configfile(Dockerfile) for each application, that has instructions on how to run the app. with this docker knows how to run our application.

### Dockerfile

Let's add Dockerfile to numbers app. here is`arithmetic-graphs/numbers/Dockerfile`
``` text
FROM python:3.7

WORKDIR /numbers

EXPOSE 80

COPY . .

RUN pip install -r requirements.txt

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "80"]
```
- We will learn about first 4 lines little later
- In 5th line, we tell docker about how to install requirements
- In 6th line we tell docker about how to run the application.

Add web app config at `arithmetic-graphs/web/Dockerfile`
``` text
FROM python:3.7

EXPOSE 8501

WORKDIR /web

COPY . .

RUN pip install -r requirements.txt

CMD ["streamlit", "run", "main.py"]
```
- This is similar to numbers app config, with command changes.

### About Docker

* As we know virtual environments provides isolated environment at python level, where as docker provides virtual isolated environment at operating system level in a very smart way using container technology.
* By adding docker file to an application, we not only managing our application but also the whole system infrastructure where our application runs. That simple 6-7 line configuration is enough to setup our application with the infrastructure.
* When we ask Docker to run the application, it goes through the dockerfile instructions and creates the whole virtual system for us and runs our application with in it.
* (little more details with anology)Dockerfile is a recipe to create Docker Image. Docker image is docker's way of storing Dokerfile instructions, those can be used to create instances of the image (final dish). You can create multiple instances from the image, where every instance is virtual os running our application.
* In docker's language the instance is nothing but the container.
 

### Back to our project
now we know that docker builds an image from the Dockerfile given, and we can run application using docker image.

Let's see how these instructions used by docker
* `FROM python:3.7` - here  python:3.7 is a docker image that docker gets from dockerhub. This one line is enough to create OS infrastructure with python 3.7 installed.
* `EXPOSE 8501` - As docker is creating virtual OS for our application, we tell docker to expose port 8501 out.
* `WORKDIR /web` - this sets working directory for any commands follows this Dockerfile.
* `COPY . .`  - copies the files from current directory into work directory in virtual os.
* `RUN` - runs the command in virtual os
* `CMD` - This will be used to start the application

Docker has commands like `docker build` to build the image from docker file and `docker run` to create instance(container) from the image. We can use these commands and can run all our applications, or otherwise we can configure all of these applications in one file and ask docker-compose to run all those at once.

### Docker compose
Now in our project we have two applications. without docker compose you can build images for both and can run containers, one for web and one for numbers app. with docker-compose you can configure how to buid all your images at one place and can have a single interface to run all your applications.

here is arithmetic-graphs/docker-compose.yml we need to have
``` text
version: '3'
services:
  api-service:
    build: numbers
    ports:
      - "8080:80"
  web-service:
    build: web
    ports:
      - "80:8501"
    environment:
      NUMBERS_URL: http://host.docker.internal:8080/numbers
```
- Here we specify how to build images for our applications and settings need to run and access our applications.
- There are two services named `api-service` and `web-service`.
- `api-service` builds image on numbers and `web-service` builds image on `web`.
- `ports` configured as host-port:container-port, that says where the app is exposed in host machine
- `environment` is environment configuration that needs to be set in container.

Now, we need one command `docker-compose up` to bring all our applications up and running.

here we have seen very little on how to use docker compose. If you want to bring database, caching etc to your project, all you need to do is add few more lines in configuration. 

Go through the [docker docs](https://docs.docker.com/get-started/) to get deeper knowledge about docker.

Get the codebase of `arithmetic-graphs` project from [here](https://github.com/leela/arithmetic-graphs).







