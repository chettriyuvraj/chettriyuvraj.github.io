## Rant

If you have ever run through a university-level course on a programming language (say Java), you have likely come across the concept of interfaces.

 Even more likely is the fact that you have been given an example akin to:

```
interface Bicycle {
    void speedUp(int increment);
}

class NormalBicycle implements Bicycle {
    int speed = 0;
    void speedUp(int increment) {
        speed += increment
    }
}

class GearBicycle implements Bicycle {
    int speed = 0;
    void speedUp(int increment) {
        changeGear();
        speed += increment
    }
}
```

 Can we be critical of such an example? An introductory programming course is like being hit with a barrage of ideas all at once, trying to piece things together. With that context in mind, such an example does kinda-sorta touch upon the concept of _single interface, multiple implementations_.

The trouble lies come the end of the course. Having never been exposed to how interfaces are used in real systems, the exam sheets end up being filled with examples like:
- Birds flying()
- Vehicles starting_engine()
- Cats attacking()

I mean all of these are logical, if somewhat convoluted deductions from the _Bicycle_ example, right?

## Thoughts

- If all you are left with at the end of a course/book/tutorial are these examples, is your understanding any further than someone who doesn't code? Present the _Bicycle_ example to any _smart_ person and they can easily deduce the other examples. 
- Conversely, when you come across a real-world opportunity to use an interface, it won't click. You end up with a problem to which you have seen the solution, but still don't know how to solve. 
- Heck, even that would be a good programming assignment - NOT using an interface and then being shown how one would naturally fit
- You become like that child who rote learnt multiplication tables. You know 6 x 7, but blank out on 7 x 6

Considering the above, I would think that how a lot of people understand interfaces is when they come across a well-written one, and life suddenly makes sense all at once.

Why not do exactly that, then? 

We will use a fairly routine task as the basis for our understanding: mocking responses from an HTTP server to a client sending requests.
## Real Life Scenario

- You were asked to create an HTTP client that hits a server endpoint and gets the response
- _"Easy peasy breezy"!_
- You created a client. Grabbed the response. Ran it through a bunch of handlers depending on the HTTP status code.
- Now you want to test it without hitting the real endpoint because, let's say they charge 1000$ per hit

Solution 1: You can create an actual server using the _NewServer()_ function defined in the _httptest_ package, which is a reasonable choice.

Solution 2: Cleverly make use of an existing Golang interface: the _RoundTripper_. Of course we're going to use this solution! What the hell was I on about otherwise? Trying to look smart with interfaces and all..

We will proceed in a bottom-up fashion, starting from the _RoundTripper_ interface itself and building our way to the top of the pyramid: using _RoundTripper_ to mock an HTTP server response.

## RoundTripper and interfaces in Go 

The _[net/http](https://pkg.go.dev/net/http)_ package in Go's standard library defines the _RoundTripper_ interface as

```
type RoundTripper interface {
    RoundTrip(*Request) (*Response, error)
}
```

It is an interface with a single method _RoundTrip_ that takes in a _Request_ and returns a _Response_. 

For those of you unfamiliar with Go's interface mechanism. Go uses the notion of _[Duck Typing](https://en.wikipedia.org/wiki/Duck_typing#:~:text=In%20computer%20programming%2C%20duck%20typing,used%20for%20a%20particular%20purpose.)_ for interfaces. To match structs to interfaces, it uses the the _duck test_. According to the _duck test_, _if it walks like a duck and it quacks like a duck, then it must be a duck_. What does this mean? Any struct that has a _RoundTrip_ method is automatically a _RoundTripper_!

How is this different from other languages? Take Java for example: objects have to explicitly declare the interfaces they implement i.e. _class NewObject implements Roundtripper_. Interfaces in Go are implicit - no need for such declarations. 

## HTTP Requests in Go

The first step was to understand the _RoundTripper_ interface.

Next, we will look at how client requests are sent to the server in Go. Our goal is to mock an HTTP server response, but what for? To test our client requests! Understanding how client requests are sent will be key to this goal.

Go's _[net/http](https://pkg.go.dev/net/http#hdr-Clients_and_Transports)_ documentation provides the following example along with some commentary 
```
For control over HTTP client headers, redirect policy, and other settings, create a Client:

client := &http.Client{
    CheckRedirect: redirectPolicyFunc,
}
resp, err := client.Get("http://example.com")


For control over proxies, TLS configuration, keep-alives, compression, and other settings, create a Transport:

tr := &http.Transport{
    MaxIdleConns:       10,
    IdleConnTimeout:    30 * time.Second,
    DisableCompression: true,
}
client := &http.Client{Transport: tr}
resp, err := client.Get("https://example.com")
```

Simple enough, right?

The way to execute an HTTP request is simply to create a client. Define (Optionally) a transport. Execute your request using the client's baked in methods such as client.Get() or client.Post()

## Clients and Transports: A closer look

 Let's take a closer look at the _Client_ and the _Transport_ struct

Our _[Client]_(https://pkg.go.dev/net/http#Client) struct is defined as

```
type Client struct {
    Transport RoundTripper
    CheckRedirect func(req *Request, via []*Request) error
    Jar CookieJar
    Timeout time.Duration
}
```

Notice anything? _Transport_ is a _RoundTripper_. Interesting!

That means it is implementing the _RoundTrip_ method. Let's us confirm this: the _Transport_ struct is located in the _transport.go file_. A quick search for _RoundTrip_ gives us no method definitions for _RoundTrip_. Weird.. 

Try it yourself: Go to [transport.go](https://cs.opensource.google/go/go/+/refs/tags/go1.22.0:src/net/http/transport.go), try to find the method definition for _RoundTrip_. No luck!

We do have a _roundTrip_ though (hmm):

![](../assets/images/Pasted%20image%2020240216161048.png)

A bit more digging and we find _RoundTrip_ here: a _roundtrip.go_ file with only one method definition: for _Transport.RoundTrip_ which calls the internal _Transport.roundTrip_ we saw above. 

![](../assets/images/Pasted%20image%2020240216161733.png)
Why this weird structuring? I have absolutely no clue! Like any other codebase, Go's source code has rabbit holes you don't want to get into. Hit me up if someone does find the commit and the reason for this, though!

So far, we have:
1. Seen that _Client_ has a field called _Transport_, which is a _RoundTripper_
2. Confirmed that _Transport_ is a _RoundTripper_ by wading through Go source code
3. Understood that real codebases can be weird.

## Clients and Transports: An even closer look

Let's continue digging to see if we can make sense of what _Clients_ and _Transports_ actually are

The [Client struct documentation](https://pkg.go.dev/net/http#Client) additionally states:

```
A Client is an HTTP client. Its zero value (DefaultClient) is a usable client that uses DefaultTransport.

A Client is higher-level than a RoundTripper (such as Transport) and additionally handles HTTP details such as cookies and redirects.
```

The second line is particularly interesting. _A Client is higher level than a RoundTripper (such as a Transport) and additionally handles HTTP details such as cookies and redirects._ This seems to imply that _Client_ is simply a wrapper around the _Transport_ with additional options to control certain settings.

Let's find out if this is true. We'll go all the way into _Client.Get()_ and see how an HTTP request is actually made.

_Client.Get()_ seems to be a wrapper around _Client.Do()_. This must also apply to other methods like _Client.Post()_ (go and check):
![](../assets/images/Pasted%20image%2020240216163846.png)

_Client.Do()_ calls into the internal _client.do()_, which looks like a hot mess:
![](../assets/images/Pasted%20image%2020240216164016.png)

Amidst all the muck, we can make an educated guess that _Client.send()_ is executing the request:
![](../assets/images/Pasted%20image%2020240216164245.png)

_Client.send()_ does a small couple of things, but we are interested in the _send()_ function that it calls. It also passes to the function, our _Transport_. This is getting interesting!
![](../assets/images/Pasted%20image%2020240216164458.png)

_send()_ again is doing a bunch of stuff, but the comment tells us we are on the right track. Somewhere around here there is an HTTP request being sent:
![](../assets/images/Pasted%20image%2020240216164723.png)

Look closely and.. it's our _Transport_! _rt.RoundTrip_ seems to be the call that is executed to get the response. We can't follow the trail anymore because _RoundTrip_ is an interface method, remember? Every _RoundTripper_ will have it's own internal implementation of _RoundTrip_ 
![](../assets/images/Pasted%20image%2020240216164912.png)

What did we learn?

- _Client_ is simply a wrapper around _Transport_, which is where the actual magic of an HTTP request happens.
- The _RoundTripper_ interface and its _RoundTrip_ method is basically an abstraction which defines the concept of sending a request and getting a response.

I am hoping you are beginning to see the outline of why interfaces are powerful..

## Mocking an HTTP server response

If you haven't joined the dots already. Here's how you can test your HTTP Client Requests using the RoundTripper interface.

Here's what we have. A client making HTTP requests, getting the response, and performing some manipulations on the response:
```
type CustomClient struct {
    client http.Client
}

func NewCustomClient(rt http.RoundTripper) *CustomClient {
    return &CustomClient{
        client: http.Client{
            Transport: rt,
            Timeout: TIMEOUT * time.Second,
        },
    }
}

func (f *CustomClient) MakeRequest(URL string) (string, error) {
    fmt.Fprintf(os.Stderr, "\n\nMaking new request to %v...", URL)

    resp, err := f.client.Get(URL)
    if err != nil { /* request itself returns an error */
        return "", ClientUnexpectedError
    }
    defer resp.Body.Close()

    switch sc := resp.StatusCode; sc {
        case http.StatusOK:
            return f.handle200(resp)
        case http.StatusTooManyRequests:
            return f.handle429(resp)
        default:
            return f.handleUnexpectedResponse(resp)
    }
}

/**** HELPERS ****/
func (f *CustomClient) handle200(resp *http.Response) (string, error) {
    body, err := io.ReadAll(resp.Body)

    // can do all sorts of manipulations on this response..
    // extract vowels.. 
    // convert this response to lowercase..
    // in our case, we will only convert it to uppercase..

    return upperCaseBody, nil
}
```

Our server returns a 200 status code and the response _"This is Sparta!"_ Our handler will convert this to uppercase. Just to restate: this handler logic is what we are looking to test, which is why we need to mock the server response.

First, define a custom RoundTripper

```
type MockRoundTripper struct {
    t *testing.T
    responses []*http.Response
    reqIndex int
}

func NewMockRoundTripper(t *testing.T) *MockRoundTripper {
    return &MockRoundTripper{
        t: t,
    }
}

func (m *MockRoundTripper) StubResponse(statusCode int, header *http.Header, body string) {
    resp := &http.Response{
        StatusCode: statusCode,
        Proto: "HTTP/1.1",
        ProtoMajor: 1,
        ProtoMinor: 1,
        Header: *header,
        Body: io.NopCloser(strings.NewReader(body)),
    }
    m.responses = append(m.responses, resp)
}

func (m *MockRoundTripper) RoundTrip(req *http.Request) (*http.Response, error) {
    m.t.Helper()
    require.Less(m.t, m.reqIndex, len(m.responses), fmt.Sprintf("error: number of requests %d, number of responses %d", m.reqIndex, len(m.responses)))
    resp = m.responses[m.reqIndex]
    m.reqIndex += 1
    return resp, nil
}

```

Explanation:
- _MockRoundTripper_: Has 3 fields, _t_ stores the testing state from Go's test package; _responses_ stores the mock responses that we want our server to return; _reqIndex_ stores the current request index i.e. first response will be returned for the first request, second for the second and so on.

- _StubResponse()_: Accepts _statusCode_, _headers_ and the _body_ for the response we want from our server. It then stuffs it into an http.Response object and adds it to our response array.

- _RoundTrip_: Simply returns the response depending on the reqIndex and increases the reqIndex.


Finally, the test for our handler looks like

```
func TestHandle200(t *testing.T) {
    body := "This is Sparta!"
    mockRoundTripper := testutils.NewMockRoundTripper(t)
    f := NewFetcher(mockRoundTripper)
    mockRoundTripper.StubResponse(200, &http.Header{}, body)
    handledResp, err := f.MakeRequest("http://www.dummyurl.com") // URL doesn't matter 
    require.NoError(t, err)
    require.Equal(t, handledResp, strings.ToUpper(body))
}
```

Ba Dum Tss!

## Putting the pieces together

Why did I wade into the muddy waters of Go's standard library?

To show how an actual interface, the _RoundTripper_ interface was nicely wrapped around and then used deep inside the wrapper to perform quite a simple task: execute an HTTP request.

We then defined a custom implementation for this interface and used it to perform a routine task: mock an HTTP server response. This shows the actual meaning of the phrase _one interface, multiple implementations._

Where do you go from here? Try going through some more of Go's interfaces such as _io.Reader_ and _io.Writer_ and how they can be used. Consider this scenario: how would you use another one of Go's interfaces to test a CLI program that outputs to _stdout_

## Resources

- The _MockRoundTripper_ is borrowed from [Daniel Wagner Hall's](https://github.com/illicitonion) implementation [here](https://github.com/CodeYourFuture/immersive-go-course/blob/impl/output-and-error-handling/projects/output-and-error-handling/testutils/mock_round_tripper.go). It reads like poetry.
