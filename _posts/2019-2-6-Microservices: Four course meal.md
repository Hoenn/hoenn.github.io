---
toc: true
toc_label: "mcrosvc"
toc_icon: "beer"
toc_sticky: true
thumbnail: /assets/images/go/gopher.png
---
Creating an example microservice system first hand

## Project structure
The idea behind this article is to explore the _basics_ of writing a small handful of microservices that can be built upon to explore various topics like [rpc](https://microservices.io/patterns/communication-style/rpi.html), [event-sourcing](https://microservices.io/patterns/data/event-sourcing.html), [instrumentation](https://microservices.io/patterns/observability/application-metrics.html) and the like. The implementation details will be light but they will hopefully employ powerful real world technology. The code for the project lives on [github](https://github.com/Hoenn/mcrosvc) but the code specific to _this_ post can be found on this [branch](https://github.com/Hoenn/mcrosvc/tree/post1)

For this project the plan is to create a `web-api` and `userdb` service. The directory structure will be a "mono-repo" where we keep the files for both of these applications in the same repository

```
mcrosvc/
    proto/
        main.proto
    web-api/
    |   README.md
    |   cmd/
    |       web-api-server/
    |           main
    |   pkg/
    |       some/
    |           modules/
    |   bin/
    |       web-api-linux-amd64
    udb/
     ...
```
`web-api` will be an HTTP API to support CRUD operations for `User`s. It will host an HTTP server and rewrite requests as `gRPC` requests to `userdb` which will perform database transactions to CRUD a `mysql` database.

This is more of an exercise than a real application, as `web-api` here is superfluous. If however, there was more complicated edge logic to be bound somewhere, perhaps `web-api` would see some major benefit of being a separate service. It could also serve multiple underlying services _for better or worse_.

I'm not an expert on any of the following, take any code or design with a grain of salt
{: .notice--info} 

## proto definitions
Since both applications will be making use of our generated protobufs, I'll define a simple message that we expect to be passed between them in requests and responses. I don't think I would normally suggest using a protobuf to "define a type" for our consumer code, but in this case it will be a simple structure and clear that both `web-api` and `udb` will utilize it.

For the very start, and for a sanity check setting up our structure: let's define a user message in `proto/main.proto`
```protobuf
syntax="proto3";

package proto;

message User {
    string name = 1;
    int32 age =2;
    int32 user_num=3;
}
```
And to generate `main.pb.go` we'll use

`protoc --go_out=. *.proto`

Awesome. Now we've generated a ton of helper methods and can start building some functionality relying on the `User` struct we can expect to be piped through. This protobuf definition is likely to change as we go through development so we just need to make sure we include `protoc` in our build automation.

## Building out udb

### Starting with a basic proto
It's easier to start with udb since it has no dependencies on other services. We'll create a `main.go` that initializes a struct with database credentials. For now we'll just build out the CRUD operations and setup the real gRPC structure later.

Let's start by stubbing out an interface and struct for our API

```go

// udb/pkg/db/interface.go
package db

import "github.com/hoenn/mcrosvc/proto"

type UserDB interface {
	CreateUser(*proto.User) (int64, error)
	GetUser(int32) (*proto.User, error)
	DeleteUser(int32) error
}

// udb/pkg/db/api.go
package db

import (
	"database/sql"
	_ "github.com/go-sql-driver/mysql"

	"github.com/hoenn/mcrosvc/proto"
)

type UserAPI struct {
	db *sql.DB
}

func NewUserAPI(database *sql.DB) *UserAPI {
	return &UserAPI{
		db: database,
	}
}

func (d *UserAPI) CreateUser(u *proto.User) (int64, error) {
	//TODO
	return nil
}
func (d *UserAPI) DeleteUser(u int32) error {
	return nil
}
func (d *UserAPI) GetUser(u int32) (*proto.User, error) {
	return nil
}
```
This will serve as the backbone of our API. We'll add a layer in `pkg/server` to be responsible for implementing the protobuf API and `UserCreateRequests` etc. as we delve further.

Putting these pieces together, our `main.go` entry point starts to take shape.
```go
// udb/cmd/udb-server/main.go
package main

import (
	"database/sql"
	"fmt"
	_ "github.com/go-sql-driver/mysql"

	"github.com/hoenn/mcrosvc/udb/pkg/db"
)

func main() {
	dbConn := &DBConn{
		Username: "username",
		Password: "password",
		IP:       "127.0.0.1",
		Port:     "3306",
	}

	fmt.Println("Setting up DB")

	d, err := sql.Open("mysql", dbConn.Format())
	if err != nil {
		panic(err.Error())
	}

	defer d.Close()

	udb := &UDBServer{
		DB: db.NewUserAPI(d),
    }
    
    //TODO:This is where a blocking loop will go
}

type UDBServer struct {
	DB db.UserDB
}

type DBConn struct {
	Username string
	Password string
	IP       string
	Port     string
}

func (d *DBConn) Format() string {
	return fmt.Sprintf("%s:%s@tcp(%s:%s)",
		d.Username,
		d.Password,
		d.IP,
		d.Port)
}
```
### Defining our gRPC service API
At this point `udb` is stubbed out in pretty decent shape. The implementation details for the CRUD operations is pretty simple so I'll skip them for now. Next we'll define a gRPC service in `proto/main.proto`

```protobuf
message CreateUserRequest {
    User user=1;
}
message CreateUserResponse {
    User user=1;
}

message GetUserRequest {
    int32 user_num=1;
}

// ...

service UDBAPI {
    rpc GetUser (GetUserRequest) returns (GetUserResponse);
    rpc CreateUser (CreateUserRequest) returns (CreateUserResponse);
    rpc DeleteUser (DeleteUserRequest) returns (DeleteUserResponse);
}
```
When we regenerate our `main.pb.go` file we see the new messages but don't see any 
changes related to `service UDBAPI`. This is because protobuf doesn't contain an RPC implementation by default, but we can get that feature with a plugin. From now on when we regenerate our `pb.go` we'll use the `grpc` plugin

`protoc --go_out=plugins=grpc:. *.proto`

Now our `main.pb.go` file includes 
```go
type UDBAPIServer interface {
	GetUser(context.Context, *GetUserRequest) (*GetUserResponse, error)
	CreateUser(context.Context, *CreateUserRequest) (*CreateUserResponse, error)
	DeleteUser(context.Context, *DeleteUserRequest) (*DeleteUserResponse, error)
}
```
We'll implement this interface in `pkg/server/server.go` and extract the properties from the request to send to our non rpc oriented API in `pkg/db/api.go`

### Quick sidetrack to simplify building our project
For the sake of language agnosticism we'll quickly build out a `Makefile` to quickly build our `go` binary and regenerate our protobufs. This will end up being extendable and we'll automate any other tasks that we might want to do in build, deploy, or test phases.

We'll use seperate Makefiles for both services as well as some shared Makefile for the project that both services might take advantage of.
```makefile
# mcrosvc/...
generate-grpc:
	# To be used from within /udb and /web-api
	protoc -I ../proto --go_out=plugins=grpc:../proto ../proto/*.proto
# udb/...
include ../Makefile
BINARYNAME=udb-server
BINARYPATH=target
BINARY=${BINARYPATH}/${BINARYNAME}

SRCPATH=cmd/udb-server/main.go

build:
	go build -o ${BINARY} -v ${SRCPATH}

run:
	./${BINARY}
```
Now from within `udb/` we can run `make build` and we'll have a binary at `udb/target/udb-server`. We could easily integrate tests or additional linting into our build process using this Makefile.

### udb gRPC refactor
Now to stub out the gRPC layer in our application. This will be responsible for unwrapping incoming gRPC requests, sending it off to our database API, then packing up a response to be returned.
```go
// pkg/server/server.go
package server

import (
	"context"

	"github.com/hoenn/mcrosvc/proto"
	"github.com/hoenn/mcrosvc/udb/pkg/db"
)

//This was previously defined in cmd/udb-server/main.go
type UDBServer struct {
	DB db.UserDB
}

func (s *UDBServer) GetUser(ctx context.Context, in *proto.GetUserRequest) (*proto.GetUserResponse, error) {
	//TODO
	return nil, nil
}
// ...
}
```
Inside these stubs we'll extract the information we need to pass on. Here's an example of Create and it's matching Create method from `db.UserDB`

```go
// pkg/server/server.go
func (s *UDBServer) CreateUser(ctx context.Context, in *proto.CreateUserRequest) (*proto.CreateUserResponse, error) {
	u := in.GetUser()
	id, err := s.DB.CreateUser(ctx, u)
	if err != nil {
		return nil, err
	}
	return &proto.CreateUserResponse{
		User: &proto.User{
			UserNum: int32(id),
			Age:     u.Age,
			Name:    u.Name,
		},
	}, nil
}

// pkg/db/api.go
func (d *UserAPI) CreateUser(ctx context.Context, u *proto.User) (int64, error) {
	res, err := WithAutomaticCommit(ctx, d.db, func(tx *sql.Tx) (sql.Result, error) {
		res, err := createInsertUserQuery(u).RunWith(tx).ExecContext(ctx)
		if err != nil {
			return nil, errors.Wrap(err, "could not insert user")
		}
		return res, nil
	})
	id, err := res.LastInsertId()
	if err != nil {
		return -1, errors.Wrap(err, "could not get id from insert")
	}

	return id, err
}
```
This seems like a good way to separate our concerns with gRPC requests and responses and our actual database API. 

### Quick detour to set up mysql and docker-compose
Now that we've implemented the stubbed methods of both our gRPC API and our backend database API, we can make a comfortable setup to run `mysql` in a docker container. To do that we'll need a minimal `docker-compose.yml` config. Well also set up a minimal configuration for running the `udb-server` binary along with the database.

```yaml
version: '3.3'
services:
  db:
    image: mysql:5.7
    environment:
      MYSQL_DATABASE: 'usersdb'
      MYSQL_USER: 'udb'
      MYSQL_PASSWORD: 'sekret'
      MYSQL_ROOT_PASSWORD: 'sekret'
    command: --init-file /data/app/init.sql
    volumes:
      - ./udb/structure.sql:/data/app/init.sql
    ports:
      - '3306:3306'
    expose:
	  - '3306'

  udb:
    image: golang:latest
    entrypoint: /bin/wait-for-it.sh db:3306 -- /bin/udb-server
    environment:
      DB_PASSWORD: 'sekret'
      DB_USER: 'udb'
      DB_NAME: 'usersdb'
      DB_ADDRESS: 'db:3306'
    volumes:
      - ./udb/target/udb-server:/bin/udb-server
	  - ./common/wait-for-it.sh:/bin/wait-for-it.sh
	depends_on:
	  - db

```
We can now build locally and run `udb-server` with the database credentials as environment variables. Replacing our hard coded values with `os.Getenv("DB_USER")`.

### Adding the gRPC server
Our gRPC API has been fleshed out, our database API is complete, we now have a mysql connection to a database running in a container, what's next is to actually start our gRPC server to accept requests.

```go
func startGRPC(udb *server.UDBServer) {
	lis, err := net.Listen("tcp", fmt.Sprintf("localhost:%s", os.Getenv("GRPC_PORT")))
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	grpcServer := grpc.NewServer()
	proto.RegisterUDBAPIServer(grpcServer, udb)
	//This is a blocking call
	grpcServer.Serve(lis)
}
```
We can add a call to `startGRPC` at the end of our main function, if we want to start other routines before blocking we'll have to `go startGRPC` instead of a normal function call. We'll also need to amend `docker-compose.yaml` with the following
```yaml
udb...
    environment:
      GRPC_PORT: '50052'
    expose:
      - '50052'
```

And a quick example of the client code we might use, at least if we're writing a go client.
```go
conn, err := grpc.Dial("localhost:50052", grpc.WithInsecure())
if err != nil {
	//Actually handle this error!
	panic(err.Error())
}
defer conn.Close()
c := proto.NewUDBAPIClient(conn)
resp, err := c.CreateUser(ctx, &proto.CreateUserRequest {
//...
```

## Conclusion
At this point the `udb` service is looking good. It's providing a gRPC API backed by its own API which will allow us to keep our direct-to-database code lean and write unit tests both with and without wrapping parameters in Requests. When building out `web-server` we can choose any language that we can `protoc --somelang_out= *.proto"` and get the benefit of a shared set of constructs and contracts between our applications. 

In a follow up post we'll implement `web-api` using some language so that we have a real rpc client and a more complete system, even with such a small scope.