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

- テスト用のREADMEをダウンロードして、HTMLに変換

```bash
❯ curl -o README.test.md -L "https://raw.githubusercontent.com/golang/go/c16c2c49e2fa98ae551fc6335215fadd62d33542/README.md"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1455  100  1455    0     0   3965      0 --:--:-- --:--:-- --:--:--  4030

❯ curl --data-binary @README.test.md http://localhost:8080/render
<h1>The Go Programming Language</h1>
<p>Go is an open source programming language that makes it easy to build simple,
reliable, and efficient software.</p>
...
```

## プロファイリング

- プロファイルを集めて、PGOを使って再ビルドしてみる
- `net/http/pprof`をimportすれば、`/debug/pprof/profile`のエンドポイントを追加してCPUのプロファイルをダウンロードできる

ここではロードを生成する簡単なプログラムを使って、プロファイルを集める (サーバーは起動したまま)

```bash
❯ go run github.com/prattmic/markdown-pgo/load@latest
go: downloading github.com/prattmic/markdown-pgo v0.0.0-20230831075821-97e1a165aa5f
```

別ターミナルを開いて、プロファイルをダウンロードする

```bash
❯ curl -o cpu.pprof "http://localhost:8080/debug/pprof/profile?seconds=30"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 16018    0 16018    0     0    531      0 --:--:--  0:00:30 --:--:--  3298
```

プロファイルを集めたら、ロードを生成するプログラムを終了させる

## プロファイルを使用する

- mainパッケージに`default.pgo`という名前のプロファイルを見つけたら自動的にPGOが有効になる
  - `pgo`フラグでPGOに使用するプロファイルのパスを指定して、`go build`に渡すこともできる
- プロファイルはrepositoryにコミットすることをお勧めする

ビルドしてみる

```bash
❯ mv cpu.pprof default.pgo

❯ go build -o markdown.withpgo.exe
```

PGOが有効になっているかチェックする

```bash
❯ go version -m markdown.withpgo.exe | grep pgo
markdown.withpgo.exe: go1.21.5
        build   -pgo=/Users/kanata-miyahana/repos/output-docs/golang/pgo/example/default.pgo
```

## 評価

- ロードを生成するプログラムのベンチマークバージョンを使ってPGOの効果を見てみる

PGOを有効にしてないサーバーを起動する

```bash
❯ ./markdown.nopgo.exe
2023/12/14 21:13:10 Serving on post 8080...
```

別ターミナルを起動してベンチマークをとる

```bash
❯ go get github.com/prattmic/markdown-pgo@latest
go: added github.com/prattmic/markdown-pgo v0.0.0-20230831075821-97e1a165aa5f

❯ go test github.com/prattmic/markdown-pgo/load -bench=. -count=40 -source $(pwd)/README.test.md > nopgo.txt
```

次はPGOを有効したサーバーを起動する

```bash
❯ ./markdown.withpgo.exe
2023/12/14 21:18:34 Serving on post 8080...
```

別ターミナルを起動してベンチマークをとる

```bash
❯ go test github.com/prattmic/markdown-pgo/load -bench=. -count=40 -source $(pwd)/README.test.md > withpgo.txt
```

### 結果を比較してみる

```bash
❯ go install golang.org/x/perf/cmd/benchstat@latest
go: downloading golang.org/x/perf v0.0.0-20231127181059-b53752263861
go: downloading github.com/aclements/go-moremath v0.0.0-20210112150236-f10218a38794

❯ benchstat nopgo.txt withpgo.txt
goos: darwin
goarch: amd64
pkg: github.com/prattmic/markdown-pgo/load
cpu: Intel(R) Core(TM) i9-9880H CPU @ 2.30GHz
        │  nopgo.txt  │            withpgo.txt             │
        │   sec/op    │   sec/op     vs base               │
Load-16   172.3µ ± 0%   172.9µ ± 1%  +0.36% (p=0.027 n=40)
```
