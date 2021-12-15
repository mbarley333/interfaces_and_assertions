# using interfaces and assertions in Go

WARNING: this a write up on using interfaces and assertions.  The scenario below is covered a different way by official go code that
was discovered upon closer review.  The main point is how to use interfaces and type assertions to solve a problem.

I encountered an issue trying to trap the following error

```bash
context deadline exceeded (Client.Timeout exceeded while awaiting headers)
```

A rushed google seach yielded some logic to trap the context deadline error:

```bash
// In hindsight, this only works when setting a context for timeouts
// I had originally not used a context and relied on the HTTPClient timeout setting: HTTPClient:  &http.Client{Timeout: 10 * time.Second},
// does not trap errors
if errors.Is(err, context.DeadlineExceeded) {
	log.Println("ContextDeadlineExceeded: true")
}
```

However, the context deadline exceeded error wasn't being caught by the logic above.  Hmmmmmmmm...



The first step was to find out what was throwing the error.  In this case, it was this block of code:

```bash
resp, err := l.HTTPClient.Do(request)
	if err != nil {
		return 0, err
	}
```
  
Next, we needed to determine the exact error that was being returned if it wasn't the context.DeadlineExceeded type.

```bash
resp, err := l.HTTPClient.Do(request)
fmt.Printf("%#v", err)
if err != nil {
		return 0, err
	}
```

The print statment actually yields:  
```bash
&url.Error{Op:"Head", URL:"http://localhost:8080", Err:(*http.httpError)(0x14000094000)}2021/12/15 07:32:58 IsTimeoutError: true
```

Interesting, nothing in the print that resembles a context error message like we are seeing so we must be on a different track.  Taking a quick look at the Do method in the http client library shows a comment that any error returned is of type \*url.Error which matches above.  If we walk through the Do method, we do see the returned error value matches the \*url.Error type.

```bash
func (c *Client) Do(req *Request) (*Response, error) {
	return c.do(req)
}
```

The c.do method contains the timeout error logic:
```bash
var didTimeout func() bool
		if resp, didTimeout, err = c.send(req, deadline); err != nil {
			// c.send() always closes req.Body
			reqBodyClosed = true
			if !deadline.IsZero() && didTimeout() {
				err = &httpError{
					err:     err.Error() + " (Client.Timeout exceeded while awaiting headers)",
					timeout: true,
				}
			}
			return nil, uerr(err)
		}

...

uerr := func(err error) error {
		// the body may have been closed already by c.send()
		if !reqBodyClosed {
			req.closeBody()
		}
		var urlStr string
		if resp != nil && resp.Request != nil {
			urlStr = stripPassword(resp.Request.URL)
		} else {
			urlStr = stripPassword(req.URL)
		}
		return &url.Error{
			Op:  urlErrorOp(reqs[0].Method),
			URL: urlStr,
			Err: err,
		}
	}

```

So, now we head over to the url package and take a look at the url.Error code:

```bash
// Error reports an error and the operation and URL that caused it.
type Error struct {
	Op  string
	URL string
	Err error
}

func (e *Error) Unwrap() error { return e.Err }
func (e *Error) Error() string { return fmt.Sprintf("%s %q: %s", e.Op, e.URL, e.Err) }

func (e *Error) Timeout() bool {
	t, ok := e.Err.(interface {
		Timeout() bool
	})
	return ok && t.Timeout()
}
```

It appears that the Timeout method might be something we want to use.  Now, we can start coming up with some logic.  We know that error is an interface, so if we can assert that an error is of type \*url.Error we can then use the Timeout method to check for a timeout.


Code:
```bash
// IsTimeout performs a type assertion using *url.Error
// as the target type.  So, if err of type *.url.Error
// we can call the Timout method
func IsTimeout(err error) bool {

	var u *url.Error

	// assert type for error interface
	u, ok := err.(*url.Error)
	if !ok {
		return false
	}

	return u.Timeout()
}

func IsTimeout(err error) bool {

	var u *url.Error

	// assert type for error interface
	u, ok := err.(*url.Error)
	if !ok {
		return false
	}

	return u.Timeout()
}
```



Tests:
```bash
// very basic test to test the assert
// logic with a non *url.Error value
func TestIsTimeoutFalse(t *testing.T) {
	t.Parallel()

	want := false
	got := linkchecker.IsTimeout(errors.New(""))

	if want != got {
		t.Fatalf("want: %v, got: %v", want, got)
	}

}
// construct a test that will pass in
// pointer to url.Error with
// the Timeout helper function
func TestIsTimeoutTrue(t *testing.T) {
	t.Parallel()

	err := &url.Error{
		Err: Faketimeout{},
	}

	want := true
	got := linkchecker.IsTimeout(err)

	if want != got {
		t.Fatalf("want: %v, got: %v", want, got)
	}

}


type Faketimeout struct{}

func (f Faketimeout) Timeout() bool {
	return true
}

func (f Faketimeout) Error() string {
	return ""
}
```


