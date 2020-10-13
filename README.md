# client-inspect
[![GoDoc](https://godoc.org/github.com/petems/client-inspect?status.svg)](https://godoc.org/github.com/petems/client-inspect) [![Go Report Card](https://goreportcard.com/badge/github.com/petems/client-inspect)](https://goreportcard.com/report/github.com/petems/client-inspect)

Allows inspection of a `http.Client` connections

Heavily based on github.com/j0hnsmith/connspy

### `http` package 

A `http.Client` suitable for debugging, writes all http data to stdout.

```go
import (
  "bytes"
  "fmt"
  "io/ioutil"
  "os"
  "regexp"
  "strings"

  clientInspect "github.com/petems/client-inspect/http"
)

func main() { 

  client := clientInspect.NewClient(nil, nil)

  resp, _ := client.Get("http://example.com/")
  // ensure all of the body is read
  ioutil.ReadAll(resp.Body)
  resp.Body.Close()

  resp, _ = client.Get("https://example.com/")
  ioutil.ReadAll(resp.Body)
  resp.Body.Close()

}
```

![image](https://user-images.githubusercontent.com/1064715/95797908-8d72f780-0ce8-11eb-97d7-5086f57c5e99.png)

You can also specify the writer with `http.NewClientWriter`, which can be used to do things like redact certain fields:

```go
import (
  "bytes"
  "fmt"
  "io/ioutil"
  "os"
  "regexp"
  "strings"

  clientInspect "github.com/petems/client-inspect/http"
)

func main() { 

  buf := new(bytes.Buffer)

  client := http.NewClientWriter(nil, nil, buf)

  resp, _ := client.Get("http://example.com/")
  // ensure all of the body is read
  ioutil.ReadAll(resp.Body)
  resp.Body.Close()

  resp, _ = client.Get("https://example.com/")
  ioutil.ReadAll(resp.Body)
  resp.Body.Close()

  httpLog := buf.String()

  s := strings.Split(httpLog, "\n")

  for count, line := range s {
    rgx := regexp.MustCompile(`^(Host: )(.+)$`)
    line = rgx.ReplaceAllString(line, `$1[REDACTED]`)
    s[count] = line
  }

  fmt.Println(s)

}
```

![image](https://user-images.githubusercontent.com/1064715/95797941-a5e31200-0ce8-11eb-95ab-c0adaa3f330d.png)

For more complex redaction purposes, I've been having a lot of luck with creating an io.Writer with logrus, then using [redactrus](https://github.com/whuang8/redactrus) to redact certain parts of the logs. Plus you can do cool formatting!

```go
package main

import (
	"io/ioutil"
	"time"

	"github.com/petems/client-inspect/http"
	"github.com/sirupsen/logrus"
	"github.com/whuang8/redactrus"
	prefixed "github.com/x-cray/logrus-prefixed-formatter"
)

func main() {

	rh := &redactrus.Hook{
		AcceptedLevels: logrus.AllLevels,
		RedactionList:  []string{"^(Host: ).+$"},
	}

	log := logrus.New()

	textFormatter := new(prefixed.TextFormatter)
	textFormatter.FullTimestamp = true
	textFormatter.TimestampFormat = time.RFC822

	log.SetFormatter(textFormatter)

	log.AddHook(rh)

	client := http.NewClientWriter(nil, nil, log.Writer())

	resp, _ := client.Get("http://example.com/")
	// ensure all of the body is read
	ioutil.ReadAll(resp.Body)
	resp.Body.Close()

	resp, _ = client.Get("https://example.com/")
	ioutil.ReadAll(resp.Body)
	resp.Body.Close()

}
```

![image](https://user-images.githubusercontent.com/1064715/95857163-1c1c5e80-0d53-11eb-9748-a5232a0be94a.png)

## Background info

[https://medium.com/@j0hnsmith/eavesdrop-on-a-golang-http-client-c4dc49af9d5e](https://medium.com/@j0hnsmith/eavesdrop-on-a-golang-http-client-c4dc49af9d5e)
