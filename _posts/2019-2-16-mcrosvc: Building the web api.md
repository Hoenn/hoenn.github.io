---
toc: true
toc_label: "mcrosvc"
toc_icon: "beer"
toc_sticky: true
thumbnail: /assets/images/icons/arrow.png
---
A http server and web-api in flask for mcrosvc

The code for the project lives on [github](https://github.com/Hoenn/mcrosvc) but the code specific to _this_ post can be found on this [branch](https://github.com/Hoenn/mcrosvc/tree/post2)

## Reflection
In `mcrosvc` so far we've done a good chunk of work. It's not perfect yet and is missing tests but we'll continue on to the `web-api` before circling back. In review we have:
- grpc service written in `go`
    - proto defined `service` and `message` for requests, responses, and a simple data structure
    - internal api layer between grpc and a mysql database
    - decent internal error handling, poor logging
- docker-compose and Makefile
    - mysql container with an init file and `structure.sql`, exposing port 3306
    - udb container exposing 50052, with environment variables to connect to mysql
    - udb will always `wait-for-it.sh` until `mysql` is ready or times out

The state of the project _before_ any further work can be found on this [github branch](https://github.com/Hoenn/mcrosvc/tree/post1)

## Overview
We'll be building a simple http server that responds to the `GET, POST` and `DELETE` methods. The server will contain a grpc client, parse requests, send the requests to `udb` and respond with a status code and body fitting what was sent back from `udb`. 

## Why Flask?
Though I rarely seem to take advantage, a pro (to some) of creating grpc services is that we can create [black box](https://docs.aws.amazon.com/aws-technical-content/latest/microservices-on-aws/characteristics-of-microservices.html) microservices of different languages and have them communicate over a common protocol. Of course the same can be said about http but with grpc we get a speed advantage and we can have some type safety thanks to our protobuf. With that said I've decided to use [flask](http://flask.pocoo.org/) as an http server giving me a chance to play with it a bit more. I wrote a short post [here](../HTTP-servers-as-a-polyglot/) covering writing a basic http server in a few different languages. 


### web-api
Flask offers quite a lot of useful features for a self-professed microframework, but we'll mostly be taking advantage of its very succinct http routing via its [route decorators](http://flask.pocoo.org/docs/1.0/quickstart/#routing). Starting with `web-api/server.py`
```python
from flask import Flask

app = Flask(__name__)

@app.route('/user')
def handleUser():
    return 'Hello world'
```
We define the `/user` route and decorate a function called `handleUser`. We can run this server with `flask run`

`env FLASK_APP=server.py flask run --port=8080`

I've elected to use port `:8080` here but the flask default `:5000` will work or any other port that doesn't conflict with another currently in use. When we `curl` on `localhost:8080/user` we get our expected response.

```shell
$ curl localhost:8080/user         
Hello World
```

`curl` without being given an http method argument with `-X` will automatically use `GET` so its clear there's a default behavior on the `@app.route` decorator that will accept `GET`. Further digging around shows that the `flask` [default behavior](https://github.com/pallets/flask/blob/0b5b4a66ef99c8b91569dd9b9b34911834689d3f/flask/app.py#L1185) for HTTP methods is only `GET`, so we'll need to include the other methods, `DELETE` and `POST`, and hopefully exclude other methods like `PATCH` for now.

```python
@app.route('/user', methods=['GET', 'POST', 'DELETE'])
```

Now we can `curl -X` those three accepted methods. Realistically we'll be running different code for each method, we can get that functionality with 

```python
from flask import request
@app.route('/user', methods=['GET', 'POST', 'DELETE'])
def handleUser():
    if request.method == 'GET':
        return 'User found!'
    elif request.method == 'POST':
        return 'User created!'
    else
        return 'User deleted!'
```

If we try to `curl -X PATCH` our flask server automatically responds with a [405](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/405) error.

### Parameters
Currently our http server doesn't do a whole lot. Remembering back to our grpc API we know the following high-level description of what we'll need to send requests to our backend.

```
CreateUser :: (Name, Age) -> Maybe(User)

GetUser :: UserNum -> Maybe(User)

DeleteUser :: UserNum -> error
```

My first take on this is to use query parameters to extract our data since it's going to be the simplest to implement. A more robust solution might be to use json requests especially if we require more data to come from the browser / `curl`.

Since we've already imported `request` we can change just the body of `handleUser` to make use of query parameters, called `args` in flask.
```python
if request.method == 'GET':
    user_num = request.args.get('id')
    return "Getting user " + user_num
elif request.method == 'POST':
    user_name = request.args.get('name')
    user_age = request.args.get('age')
    return "Creating user" + user_name + ", " + user_age
elif request.method == 'DELETE':
    user_num = request.args.get('id')
    return "Deleting user " + user_num
else:
    return ('Unsupported HTTP method', 405)
```
And provide arguments with standard query strings when hitting our server

```shell
$ curl -X GET localhost:8080/user?id=5
Getting user 5
```

## docker-compose
To begin to include `web-api` into our system so far we'll need to extend our `docker-compose.yml` file. For the `udb` service we decided to build our `go` binary locally and then use a `volume` and `entrypoint` to run our grpc server. For the sake of simplicity, we'll build and run our flask server within the container. We need to define a `Dockerfile` in the `web-api/` directory first.
```Dockerfile
FROM python:3.6-alpine
ADD . /src
WORKDIR /src
RUN pip install -r requirements.txt
CMD ["python", "server.py"]
```
`requirements.txt` simple contains `flask` for now. This docker file will be used for the `build` context in `docker-compose.yml`.

```yaml
web-api:
  build: web-api
  ports: 
    - '8080:8080'
```

When we invoke `docker-compose up --build web-api`, the `python:3.6-alpine` image will be pulled (if not already local), the `mcrsovc/web-api/` directory will be mounted to `/src`, and then from `/src` our start command `python server.py` will be run. Since we specified a port `8080:8080`, our `localhost:8080` should redirect into the container, and we can repeat the same testing with `curl` as we did before.

## gRPC client
Now that we've made a mock up of our http API let's start poking the `udb` server via. gRPC. To create our protos and client we'll amend the Makefile `generate-grpc` target:
```diff
generate-grpc:
 	protoc -I ../proto --go_out=plugins=grpc:../proto ../proto/*.proto
+	python -m grpc_tools.protoc -I ../proto --python_out=../web-api --grpc_python_out=../web-api ../proto/*.proto
```
This will generate the python code representation of our protos and we can try it out in `main.py`.


```python
import grpc
import main_pb2 as proto
import main_pb2_grpc as proto_grpc
#...
def main():
    channel = grpc.insecure_channel(os.environ['UDB_ADDRESS'])

    client = proto_grpc.UDBAPIStub(channel)
    user = proto.User(name="evan", age=22)
    req = proto.CreateUserRequest(user=user)
    resp = client.CreateUser(req)
    print(resp)
```

 Let's catch up our docker-compose environment for web-api -> udb and add in the `'UDB_ADDRESS'`.

```yaml
  web-api:
    build: web-api
    environment:
      UDB_ADDRESS: 'udb:50052'
    ports:
      - '8080:8080'
    depends_on:
     - udb
```
Now we can `docker-compose up` to bring up our whole stack of `udb`, `web-api` and `db`. As soon as `web-api` starts it will create a user via. gRPC from the example code in main and we should see 
```json
user {
  name: "evan"
  age: 1
  user_num: 1
}
```
in `docker-compose logs web-api` showing a successful request.

### Updating our routes to make gRPC requests
We can remove our test code and update the http routes we defined earlier.
```python
channel = grpc.insecure_channel(os.environ['UDB_ADDRESS'])
client = proto_grpc.UDBAPIStub(channel)

@app.route('/user', methods=['GET', 'POST', 'DELETE'])
def handleUser():
    if request.method == 'GET':
        user_num = request.args.get('id')
        print("Get user " + user_num)
        return getUser(user_num)
    # ...
def getUser(id):
    req = proto.GetUserRequest(user_num=int(id))
    resp = client.GetUser(req)
    return str(resp)
```
We define `channel` and `client` globally just for convenience. The rest of the routes are easy to define, just paying attention to convert any query string arguments into the correct types that are expected in the proto definitions.

## Conclusion
gRPC is a great technology built on great technologies like HTTP/2. Being able to define service contracts via proto definitions is what I appreciate most. We generated the boilerplate for `go` and `python` here and cleaned up what could have been a pretty ugly integration between the `web-api` and `udb` servers hopefully showing that gRPC serves the polyglot use-case very well.