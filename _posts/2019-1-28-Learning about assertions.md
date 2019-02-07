---
thumbnail: /assets/images/go/gopher.png
---

In this post I'll run through a protobuf example where I learned more about type switching and assertions

## Background
In the time I've spent programming large swathes of time has been spent stuck to one language or another depending on what projects I was working on. 

My personal pie chart of time spent (not projects completed) is along the lines of Java (40%), JavaScript(20%), Go (20%), TypeScript(10%), Python(5%), Haskell (5%). 

While I haven't always had the choice of how to spend development time, picking the right tool for the job, I'm very drawn to expressive type systems and static typing. Now, at work, it's not really up to what language I'm working in and to be completely honest I wouldn't make a `case` for a `switch` anyway. 

I decided to write this post to show a couple of the stumbles I've had not thinking like a Gopher.

## Quick exercise to remember interfaces
This isn't the right place for a deep dive on how these concepts are presented in every language I've learned. But, the gist that I've held (which may be incorrect) are that they describe "what" a `thing` can do and include either no or some implementation (abstract classes).

In a language like Java you can use these to great effect to build heirarchies of types like this
```java
//Shape.java...
public interface Shape {
    float area()
}
//Circle.java...
public class Circle implements Shape {
    private float radius
    public Circle(float r) {
        this.radius = r;
    }
    //Compiler error if we don't implement area()!
    public float area() {
        return Math.pi * Math.pow(this.radius, 2);
    }
}
//Rectangle.java...
public class Rectangle implements Shape {
    private float width, height;
    public Rectangle(float w, float h) {
        this.width = w;
        this.float = h;
    }
    public float area() {
        return this.width * this.height;
    }
}
//main.java...
public void shapeToString(Shape s) {
    System.out.println("Area of shape: "+s.area())
}
```
Without any further implementation it's still possible to see the value in being able to create `ArrayList<Shape>` and know that all of the objects inside can be used in the capacity of a `Shape` (in this case call the `area` function). Similarly if we had a `abstract class Quadrilateral` it could implement `Shape` and `Rectangle` and `Square` could extend it.

A similar program in `go` might look like
```go
type Shape interface {
    float32 Area()
}

type Circle struct {
    float32 Radius
}

//Circle implements the Shape interface
func (c *Circle) Area(){
    return Math.Pi * Math.Pow(2, c.Radius)
}

type Rectangle struct {
    float32 Height
    float32 Width
}

//Rectangle implements the Shape interface
func (r *Rectangle) Area() {
    return r.Height * r.Width
}
```
The big difference here is: implicit interface implementation. This has a consequence of lacking a compiler warning (or error like `java`) when you meant to implement some interface, or thought you did implement some interface but had the `func` header slightly wrong! Still a simple enough translation.

## Getting lost in the protobuf

Where I started to get tripped up with `go`'s interfaces is when I wanted to see if an extended protobuf generated struct implemented an interface
```protobuf
//Let's pretend a Login could contain either a password or some qr code bytes
message Login {
    oneof kind{
        string pass = 1;
        bytes qrdata = 2;
    }
}
```
The code that is generated is
```go
type Login struct {
    Kind isLogin_Kind `protobuf_oneof:"kind"`
}
type Login_Pass struct {
    Pass string
}
type Login_Qrdata struct {
    Qrdata []bytes
}
```
Later define an interface and implement that method onto these structs like this:
```go 
type Formatter interface {
    Format() string
}
func(p *Login_Pass) Format() string {
    return fmt.Sprintf("Password: %s", p.Pass)
}
func(q *Login_Qrdata) Format() string {
    return fmt.Sprintf("QR data: %v", p.Qrdata)
}
```
And lastly, for this example, we'll create a LoginEmailer.
```go
type LoginEmailer struct {
    Login login
}
func (l LoginEmailer) Send() error {
    //This is an error! isLogin_Kind does not implement Formatter!!
    b := l.login.GetKind().Format() 
}
```
This is certainly not what I expected to happen. The issue here really arises from the fact that `GetKind()` does not return a `Login_Pass`, `Login_Qrdata` or some interface type that matches both. `GetKind` would be generated as
```go
func (l *Login) GetKind() isLogin_Kind {
    if l != nil {
        return l.Kind
    }
    return nil
}
```
So we have a couple of options of dealing with this. We can use a [type switch](https://tour.golang.org/methods/16)
```go
func (l LoginEmailer) Send() error {
    var b string
    switch l := l.Login.GetKind().(type) {
        case Login_Pass: b = l.Format()
        case Login_Qrdata: b = l.Format()
        default: 
            return errors.New("Login cannot be formatted")
    } //...
}
```
This solution isn't my favorite. If we make a change to our protobuf down the line to support a new `Login_Fingerprint` we'll need to update this code with another `case`. 

A cleaner way to solve this problem is to use a [type assertion](https://tour.golang.org/methods/15)
```go
func (l LoginEmailer) Send() error {
    var b string
    l, ok := l.Login.GetKind().(Formatter)
    if !ok {
        //We wouldn't expect this to happen
        return errors.New("Login does not support formatting") 
    }
    b := l.Format()
    //...
}
```
For the particular use case that had me discover type assertions, it was preferable to be able to update the protobufs with new `oneof` values without needing to recompile and deploy the application using them.

### Could Java do better?
I'm not convinced that Java would do any better at expressing this problem, particularly because of the protobuf wireformat. After poking around a bit it looks like the Java developers share [similar pain](https://github.com/protocolbuffers/protobuf/issues/2984) when it comes to how their `oneof` properties are used.

## Conclusion

This is definitely an instance where my preconceptions of protobuf generated code shined through. It was a good learning experience and the first time I've had to use a type assertion that wasn't copy pasta. The goal of trying to keep this microservice abstract enough to accept a new `oneof` entry and its formatter implementation without recompilation felt worth the trouble.