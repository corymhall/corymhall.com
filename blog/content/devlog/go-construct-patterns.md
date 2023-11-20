---
title: "Go Construct Patterns"
date: 2023-11-18T05:54:32-05:00
draft: false
---

Recently I've been working a lot with the AWS CDK in Golang and there isn't a
lot of prior art to reference. When working in TypeScript I could always just
look at the [aws-cdk](https://github.com/aws/aws-cdk) repo and find tons of
great examples. One thing that has taken me a while to develop a pattern for is
creating [constructs](https://docs.aws.amazon.com/cdk/v2/guide/constructs.html).
In TypeScript it's pretty simple, you just create a new class that extends some
other construct and you're done. In Go, it's a little more complicated.

In this post I'm going to talk through some patterns that I've discovered when
creating Go constructs. If you've discovered better ones, please let me know!

## Creating a new simple construct

I'll start off by talking about the basic example that you've probably seen
elsewhere. For the example, I'll be creating a construct with an
[HttpApi](https://pkg.go.dev/github.com/aws/aws-cdk-go/awscdkapigatewayv2alpha/v2#HttpApi)
as the core resource.

Since Go doesn't have classes, you can't just create a `MyApi` class and extend
construct like you would do in TypeScript. Instead you would create a function
that looks something like this.

```go
func NewMyApi(scope constructs.Construct, id string) constructs.Construct {
    c := constructs.NewConstruct(scope, &id)

    return c
}
```

Now I can create my resources within this function and make sure I use the
construct that I created as the `scope` that I pass in.

```go
func NewMyApi(scope constructs.Construct, id string) constructs.Construct {
    c := constructs.NewConstruct(scope, &id)

	awscdkapigatewayv2alpha.NewHttpApi(c, jsii.String("MyHttpApi"), &awscdkapigatewayv2alpha.HttpApiProps{})

    return c
}
```

That seems pretty straightforward, but I'm only returning a
`constructs.Construct`. What if I want to reference my `HttpApi` that I created?

## Constructs with properties

As soon as I want to reference that `HttpApi` outside of this construct I need
to find a way of returning something other than `constructs.Construct`. I could
take two different approaches. If I wanted this construct to really just be a
wrapper around a `HttpApi` then I could just change it to return the `HttpApi`.

```go
func NewMyApi(scope constructs.Construct, id string) awscdkapigatewayv2alpha.HttpApi {
    c := constructs.NewConstruct(scope, &id)

	return awscdkapigatewayv2alpha.NewHttpApi(c, jsii.String("MyHttpApi"), &awscdkapigatewayv2alpha.HttpApiProps{})
}
```

We'll come back to this approach a little later.

If I want this to be a more general construct or maybe an L3 construct (which
represents more than just an `HttpApi`), then the other approach would be to
just create a new `struct` and return that.

```go
type MyApi struct {
    HttpApi awscdkapigatewayv2alpha.HttpApi
}

func NewMyApi(scope constructs.Construct, id string) MyApi {
    c := constructs.NewConstruct(scope, &id)
    myApi := MyApi{}

	myApi.HttpApi := awscdkapigatewayv2alpha.NewHttpApi(c, jsii.String("MyHttpApi"), &awscdkapigatewayv2alpha.HttpApiProps{})

    return myApi
}
```

Now we can access the `HttpApi` resource outside of this function. Pretty simple
right? Not quite. In CDK anytime you create groupings of resources you typically
wrap them in a `Construct`. It allows you to more easily access the child constructs,
and do things like add a dependency that affects all child constructs.


```go
app := awscdk.NewApp()
stack := awscdk.NewStack(app, jsii.String("Stack"), &awscdk.StackProps{})
myApi := NewMyApi(stack, "MyApi")
other := NewSomeOtherConstruct(stack, "OtherConstruct")

// MyApi has a dependency on other
other.Node().AddDependency(myApi)
```

The way I've setup `MyApi`, this won't work! `myApi` is a `MyApi` struct type,
not a construct. I could just add another property to `MyApi` like this:

```go
type MyApi struct {
    HttpApi awscdkapigatewayv2alpha.HttpApi
    Construct awscdk.Construct
}

func NewMyApi(scope constructs.Construct, id string) MyApi {
    c := constructs.NewConstruct(scope, &id)
    myApi := MyApi{
        Construct: c,
    }

	myApi.HttpApi := awscdkapigatewayv2alpha.NewHttpApi(c, jsii.String("MyHttpApi"), &awscdkapigatewayv2alpha.HttpApiProps{})

    return myApi
}
```

But then I have to make sure to always reference the `Construct` property, which
is not very intuitive.

```go
other.Node().AddDependency(myApi.Construct)
```

## Construct Override

What I really want is for `MyApi` to _be_ a construct. In other languages this
might be done using `extends` or `implements`. In Go I can do this by using
`type embedding`, but as [this article](https://www.dolthub.com/blog/2023-02-22-golangs-fake-inheritance/)
points out, this is not true inheritance. Since all objects in CDK are
`interfaces` what you might end up doing is trying to embed an interface in a
struct, which as the article points out, _"is allowed but is almost never what you want."_
In CDK apps this is especially true. If we try to embed `Construct` in `MyApi`:

```go
type MyApi struct {
    awscdk.Construct
    HttpApi awscdkapigatewayv2alpha.HttpApi
}
```

We will eventually end up getting errors like this:

```shell
panic: @jsii/kernel.SerializationError: Passed to parameter construct of static method aws-cdk-lib.Stack.of: Unable to deserialize value as constructs.IConstruct
├── 🛑 Failing value is an object
│      { Construct: {} }
╰── 🔍 Failure reason(s):
    ╰─ Value does not have the "$jsii.byref" key
```

Instead we need to turn the `MyApi` struct into an interface. This means that
our struct properties also need to become interface methods.

```go
type MyApi interface {
    constructs.Construct
    HttpApi() awscdkapigatewayv2alpha.HttpApi
}
```

We can then create a private struct that implements the `MyApi` construct.

```go
type myApi struct {
    constructs.Construct
    httpApi awscdkapigatewayv2alpha.HttpApi
}

func (a *myApi) HttpApi() awscdkapigatewayv2alpha.HttpApi {
    return a.httpApi
}
```

The last piece is to use `constructs.NewConstruct_Override()`. This ensures that
it will use your `myApi` struct as the struct that internally implements the
`Construct` interface.

```go
func NewMyApi(scope constructs.Construct, id string) MyApi {
    myApi := myApi{}
    constructs.NewConstruct_Override(myApi, scope, &id)

	myApi.httpApi := awscdkapigatewayv2alpha.NewHttpApi(myApi, jsii.String("MyHttpApi"), &awscdkapigatewayv2alpha.HttpApiProps{})

    return myApi
}
```

Now you can use `MyApi` as a `Construct` and still have access to your custom
properties. The full example would look something like this.

```go
type MyApi interface {
    constructs.Construct
    HttpApi() awscdkapigatewayv2alpha.HttpApi
}

type myApi struct {
    constructs.Construct
    httpApi awscdkapigatewayv2alpha.HttpApi
}

func NewMyApi(scope constructs.Construct, id string) MyApi {
    myApi := myApi{}
    constructs.NewConstruct_Override(myApi, scope, &id)

	myApi.httpApi := awscdkapigatewayv2alpha.NewHttpApi(myApi, jsii.String("MyHttpApi"), &awscdkapigatewayv2alpha.HttpApiProps{})

    return myApi
}

func (a *myApi) HttpApi() awscdkapigatewayv2alpha.HttpApi {
    return a.httpApi
}
```

## Wrapping a L2 Construct (L2.5)

Coming back to the example earlier where I want my construct to represent my own
version of an `HttpApi` I can do something very similar to what I did with the
`Construct` example in the previous section.

```go
type MyApi interface {
    awscdkapigatewayv2alpha.HttpApi
}

type myApi struct {
    awscdkapigatewayv2alpha.HttpApi
}

func NewMyApi(scope constructs.Construct, id string) MyApi {
    myApi := myApi{}

	awscdkapigatewayv2alpha.NewHttpApi_Override(myApi, scope, jsii.String("MyHttpApi"), &awscdkapigatewayv2alpha.HttpApiProps{})

    return myApi
}
```
