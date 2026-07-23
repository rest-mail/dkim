# dkim

[![CI](https://github.com/rest-mail/go-dkim/actions/workflows/ci.yml/badge.svg)](https://github.com/rest-mail/go-dkim/actions/workflows/ci.yml)
[![Go Reference](https://pkg.go.dev/badge/github.com/rest-mail/go-dkim.svg)](https://pkg.go.dev/github.com/rest-mail/go-dkim)

DKIM ([RFC 6376](https://www.rfc-editor.org/rfc/rfc6376)) signing and
verification for Go, with zero external dependencies (standard library only).

Verification and signing both operate on the **raw** RFC 5322 message bytes —
never on a parsed or reconstructed representation — so a signature is checked
against exactly what was transmitted. Both simple and relaxed header/body
canonicalization are supported, and `rsa-sha256` (plus legacy `rsa-sha1` on the
verify path).

The package also exports its canonicalization and single-signature primitives
(`SplitMessage`, `CanonicalizeHeader`, `CanonicalizeBody`, `BuildSignedHeaders`,
`VerifySignature`, `FetchKey`, `ParseTagList`, `RemoveBValue`, `StripWSP`,
`HashBytes`) so that layered schemes such as ARC — whose ARC-Message-Signature
is structurally a DKIM-Signature — can reuse the exact same code path. See
[github.com/rest-mail/arc](https://github.com/rest-mail/arc).

## Install

```sh
go get github.com/rest-mail/go-dkim
```

## Sign

`Sign` returns the DKIM-Signature field **value**; prepend the header yourself.

```go
package main

import (
	"fmt"

	"github.com/rest-mail/go-dkim"
)

func main() {
	privateKeyPEM := "-----BEGIN RSA PRIVATE KEY-----\n...\n-----END RSA PRIVATE KEY-----\n"
	key, err := dkim.ParsePrivateKey(privateKeyPEM)
	if err != nil {
		panic(err)
	}

	raw := []byte("From: alice@example.com\r\n" +
		"To: bob@example.net\r\n" +
		"Subject: hello\r\n" +
		"Date: Thu, 23 Jul 2026 10:00:00 +0000\r\n" +
		"Message-ID: <1@example.com>\r\n" +
		"\r\n" +
		"Hello, world!\r\n")

	value, err := dkim.Sign(raw, dkim.SignOptions{
		Domain:     "example.com",
		Selector:   "default",
		PrivateKey: key,
		// HeaderCanon / BodyCanon default to "relaxed";
		// Headers defaults to from:to:subject:date:message-id.
	})
	if err != nil {
		panic(err)
	}

	signed := append([]byte("DKIM-Signature: "+value+"\r\n"), raw...)
	fmt.Printf("%s", signed)
}
```

Generate a keypair and render its DNS TXT record with `GenerateKey`,
`RecordName`, `RecordValue`, and `RecordFragment`.

## Verify

`Verify` returns one `VerifyResult` per DKIM-Signature header, in header order.
Pass `nil` for the resolver to use system DNS, or inject a `TXTResolver` (its
signature matches `net.Resolver.LookupTXT`) in tests.

```go
package main

import (
	"context"
	"fmt"

	"github.com/rest-mail/go-dkim"
)

func main() {
	raw := []byte( /* a raw message carrying a DKIM-Signature header */ )

	results := dkim.Verify(context.Background(), raw, nil)
	for _, r := range results {
		fmt.Printf("d=%s s=%s -> %s (%s)\n", r.Domain, r.Selector, r.Result, r.Reason)
	}
}
```

`r.Result` is one of `dkim.ResultPass`, `ResultFail`, `ResultNeutral`,
`ResultNone`, `ResultTempError`, or `ResultPermError` (RFC 8601 `dkim=` values).

## License

[MIT](LICENSE) © 2026 rest-mail
