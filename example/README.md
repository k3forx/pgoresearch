# Example Project

- markdownをHTMLに変換するサービスをビルドする
- markdownを`/render`にアップロードするとHTMLが返されるサービス

## Setup

```bash
❯ go version
go version go1.21.5 darwin/amd64

❯ go get gitlab.com/golang-commonmark/markdown@bf3e522c626a
```

- プログラムコード

```go
package main

import (
	"bytes"
	"io"
	"log"
	"net/http"
	_ "net/http/pprof"

	"gitlab.com/golang-commonmark/markdown"
)

func render(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "Only POST allowed", http.StatusMethodNotAllowed)
		return
	}

	src, err := io.ReadAll(r.Body)
	if err != nil {
		log.Printf("error reading body: %v", err)
		http.Error(w, "Internal Server Error", http.StatusInternalServerError)
		return
	}

	md := markdown.New(
		markdown.XHTMLOutput(true),
		markdown.Typographer(true),
		markdown.Linkify(true),
		markdown.Tables(true),
	)

	var buf bytes.Buffer
	if err := md.Render(&buf, src); err != nil {
		log.Printf("error converting markdown: %v", err)
		http.Error(w, "Malformed markdown", http.StatusBadRequest)
		return
	}

	if _, err := io.Copy(w, &buf); err != nil {
		log.Printf("error writing response: %v", err)
		http.Error(w, "Internal Server Error", http.StatusInternalServerError)
		return
	}
}

func main() {
	http.HandleFunc("/render", render)
	log.Printf("Serving on post 8080...")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

- ビルドして起動する

```bash
❯ go build -o markdown.nopgo.exe

❯ ./markdown.nopgo.exe
2023/12/14 20:39:04 Serving on post 8080...
```