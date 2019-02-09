---
toc: true
toc_label: "HTTP"
toc_icon: "beer"
toc_sticky: true
thumbnail: /assets/images/icons/server.png
---
Exploring simple HTTP servers in multiple languages

## Background
While working on [mcrosvc](../Microservices-A-four-course-meal/) I've been tempted to revisit the various tools available to make an HTTP server that plays decently nice as a gRPC client. I'm the most familiar with writing HTTP servers in `node` using [Express](https://expressjs.com/) and in `go` with the [net/http](https://golang.org/pkg/net/http/) package, I'd also like to try out some other option like [flask](http://flask.pocoo.org/).

## What are we trying to solve?
For the purposes of `mcrosvc` we're just looking for a lightweight HTTP server that we can expose multiple endpoints with. To keep things as simple as possible we'll ensure a simple `json` request body and just run the server on `localhost`. Without going into too much detail: we want a simple web server that can route requests from a browser (or likely `curl` for now), repackage the information and send it on its way to the backend. For the purposes of briefly visiting each language and (micro)framework, we'll just create a route defined for `GET`.

### Nodejs (Express)
Express makes it pretty simple to create an HTTP server in very few lines of code. We can start our server on `localhost:3000` with
```js
var express = require('express'),
    app = express()
    http = require('http').createServer(app);

app.get("/user", function(req, res) {
    //Send to the backend
    res.send("User found!")
});

http.listen(process.env.PORT || 3000, function() {
    console.log("listening on *:3000");
});
```
Once we've installed `express` with something like `npm` we can run this and open another window to test.

```
$ curl localhost:3000            
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<title>Error</title>
</head>
<body>
<pre>Cannot GET /</pre>
</body>
</html>
```
We get an error message here that `express` is handling. We get this since when we `curl localhost:3000` it's implicitly sending `GET localhost:3000/`, but only defined a `GET` route for `/user`. Fixing that in our test

```
$ curl localhost:3000/user
User found!
```

Interestingly the first parameter to `app.get` can be a regular expression should we need to take advantage of that. More blatantly useful is that we can define route parameters directly into our route string.

```js
app.get('/users/:id', function(req, res) {
    //Use req.params to access each param in the route
})
```
We can also use a callback function to handle the request
```js
app.get('/users/:id', function(req, res, next) {
    console.log("Hello")
    next()
}, function(req, res) {
    console.log("world")
    //Send to the backend
    res.Send("User found!")
})
```

### go (net/http)
`go` has a great standard library which many people love it for. There are certainly libraries with common abstractions, but our use case is simple so we can rely just on `net/http`.

```go
import (
    "net/http"
    "log"
)
func getUser(w http.ResponseWriter, r *http.Request) {
    //Send to the backend code
    w.Write([]byte("User found!"))
}
func main() {
    http.HandleFunc("/user", handleGetUser)
    err := http.ListenAndServer(":3000", nil)
    if err != nil {
        log.Println(err)
    }
}
```
If we want to access query parameters we can use the `http.Request` itself
```go
func getUser(w http.ResponseWriter, r *http.Request) {
    query := r.URL.Query()
    id := query.Get("id")
    //Send to the backend code
    w.Write([]byte("User found!"))
}
```

### Python3 (Flask)
Python is a language I've picked up over time but haven't used in production. I never felt the need to stray from languages and ecosystems I know more about but I'd like to see what the fuss with Flask is about. The code for our server, practically lifted from the [flask github](https://github.com/pallets/flask), is 
```python
from flask import Flask

app = Flask(__name__)

@app.route('/user')
def getUser():
    return 'User found!'
```

The smallest yet! Looking at the docs, extending our route with a parameter is pretty simple

```python
@app.route('/user/<id>')
def getUser(id):
    # Send to backend code
    return 'User found!'
```
Also, it seems that all HTTPs method are implicitly defined here, so we should restrict that to just `GET`
```python
@app.route('/user/<id>', methods=['GET'])
def getUser(id):
#...
```
And the last little thing would be to figure out where we can define our own port of choice. Not that port `5000` is likely to clash with anything, but I've used `3000` in the other examples. To run the server I've been using

`$ env FLASK_APP=server.py flask run`

We can define the port as an argument to `flask run`

`$ env FLASK_APP=server.py flask run --port=3000`

At this point it's unclear what all is going on under the covers in flask, but I appreciate its built in logging. The output for `curl localhost:3000` and `curl localhost:3000/user/5` below.

```
$ env FLASK_APP=server.py flask run --port=3000
 * Serving Flask app "server.py"
 * Environment: production
   WARNING: Do not use the development server in a production environment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://127.0.0.1:3000/ (Press CTRL+C to quit)
127.0.0.1 - - [09/Feb/2019 14:22:35] "GET / HTTP/1.1" 404 -
127.0.0.1 - - [09/Feb/2019 14:22:38] "GET /user/5 HTTP/1.1" 200 -
```
Flask calls itself a "micro" framework and I can see why. I'd like to learn more about Flask so I plan to use it as the basis for the `web-api` HTTP server in the [mcrosvc](../Microservices-A-four-course-meal/) series.

## Conclusion
While the main intent of this post was just to showcase how simple HTTP servers are to get going in a few languages, it's worth looking into the comparison of pros and cons. Outside of raw performance working with a language you prefer is always a boon, so I'm not placing a lot of value on "same language frontend and backend" at all. The choice of any of these tools should be on what they're useful for and if the language itself is providing you the tools you need without much fuss.