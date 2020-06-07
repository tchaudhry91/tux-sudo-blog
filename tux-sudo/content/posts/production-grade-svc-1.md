---
title: "Production Grade Web Services with Go - The Beginning"
date: 2020-06-07T09:33:15Z
categories: ["development", "go", "webservice", "tutorial"]
---

Production Grade is a term thrown around a lot these days. Every organization seems to have it's own definition for what qualifies as "Production Grade" or "Production Ready". In these series of posts, I'm going to present my take on the topic and the bare minimum of what I think qualifies under this definition. These posts assume a working knowledge of Go and are not meant for total beginners to the language.

At it's core, a webservice provides some sort of RPC (remote-procedure calls) or REST (Representational State Transfer). *REST* is often a misused term, that's a story for another time, but in short Actions define RPCs, while REST is all about the resources. For example, `/getAllUsers` screams RPC, whereas, *GET* `/users` is more of a representational state transfer. Both of these may use JSON to serialize their payload and HTTP for underlying transfer, but that does not mean they are the same thing.
This isn't going to matter a whole lot with what we're planning to do, but it's something to keep in mind.

Throughout this series, I'm going to be working on a [web service](https://github.com/tchaudhry91/rainbow) that provides two simple RPCs. I have placed strategic commits and linked them in this post to allow for "point in time" snapshot of the code at that duration. These will be indicated in the text below.

```go
hash(blob string) (hashed string) // Computes the hash and stores the value in a store.
hashReverse(hashed string) (blob string, err error) // Retrieves the "key" from the store by doing a reverse lookup.
```

These are two simple methods, one that returns the hash of a given string and another that attempts to return the reverse of a hash. If you're not very familiar with what Hashes are, the `hash` method is a simple `Put` operation and the `hashReverse` method is a simple `Get` operation. 

Hashing, by definition is a one-way process and a `hashReverse` should be meaningless. True, but nothing is stopping us *pre-computing* a lot of hashes and looking them up from a store. This is something called a **rainbow-table** and it is the reason why all hashes you store should be *salted*. Read more about it [here](https://en.wikipedia.org/wiki/Rainbow_table).

That's it. That's the entire business logic right there, and this is what we're going to start with. In fact, in this first post, we're not going to be using *ANY* of web components. We're focussing entirely on the business logic because that's what is most important. I like to build services from the **inside-out** and that's what I'm going to demonstrate. This a concept promoted by the wonderful [Go-Kit](https://gokit.io/) toolkit that I find highly effective. We're not going to use Go-Kit for our service (perhaps another series in the future), but the principles apply.

Let's bootstrap the Go repository to use modules by running:

```go
go mod init "github.com/tchaudhry91/rainbow"
```

Next, lay down some structure and create a package `service` and inside it, a file called `rainbow.go`.

As promised, we start with the business logic definition of our service and place the following interface in our `rainbow.go`:

```go
type RainbowService interface {
	Hash(blob string) (hashed string)
	HashReverse(hashed string) (blob string, err error)
}
```

That's a nice start. But, we can do better with interface composition. Check out the following and see how it helps adhere to the best practice of keeping interfaces small and composable.

```go
// Hasher is any type that can return a hash of string
type Hasher interface {
	Hash(blob string) (hashed string)
}

// HashReverser is any type that can reverse a hash and return the original blob
type HashReverser interface {
	HashReverse(hashed string) (blob string, err error)
}

// RainbowService is a service to compute hashes and lookup reverse hashes
type RainbowService interface {
	Hasher
	HashReverser
}
```

We now have meaningful interfaces and a composed service. Next up, we're going to create a concrete implementation of this interface for the SHA256 hashing algorithm.

```go
// SHA256RainbowService is a SHA256 implementation for the RainbowService
type SHA256RainbowService struct{}

func NewSHA256RainbowService() *SHA256RainbowService {
	return &SHA256RainbowService{}
}

// Hash returns the SHA256 sum for the given string
func (svc *SHA256RainbowService) Hash(blob string) (hashed string) {
	sumA := sha256.Sum256([]byte(blob))
	return hex.EncodeToString(sumA[:])
}

// HashReverse looks up the original string for the given hash
func (svc *SHA256RainbowService) HashReverse(hashed string) (blob string, err error) {
	panic("unimplemented")
}
```

This is an implementation of `Hasher`, `HashReverser` and consequently `RainbowService`. This service is however, still incomplete, we don't have a working `HashReverse` method yet due to the absence of a datastore.

To demonstrate a bit of TDD (Test Driven Development), I'm going to leave the datastore bit off for a little while and instead, let's write some tests! Create a `rainbow_test.go` and declare it to use a different package `service_test`. This way we can test our service like a black-box without having access to the internals.

Here are the two sample tests that test the two methods:

```go
func TestServiceHash(t *testing.T) {
	svc, err := getService()
	if err != nil {
		t.Errorf("Failed to get sample test service")
		t.FailNow()
	}

	type TestCase struct {
		Blob       string
		WantedHash string
	}

	cases := []TestCase{
		{"thisisastring", "572642d5581b8b466da59e87bf267ceb7b2afd880b59ed7573edff4d980eb1d5"},
		{"password", "5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8"},
		{"", "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"}, // Hash of empty-string https://www.di-mgt.com.au/sha_testvectors.html
	}

	for _, testCase := range cases {
		ReceivedHash := svc.Hash(testCase.Blob)
		if testCase.WantedHash != ReceivedHash {
			t.Errorf("Failed for blob - %s, Wanted: %s, Received: %s", testCase.Blob, testCase.WantedHash, ReceivedHash)
		}
	}
}

func TestServiceHashReverse(t *testing.T) {
	svc, err := getService()
	if err != nil {
		t.Errorf("Failed to get sample test service")
		t.FailNow()
	}

	type TestCase struct {
		Hash       string
		WantedBlob string
	}

	cases := []TestCase{
		{"572642d5581b8b466da59e87bf267ceb7b2afd880b59ed7573edff4d980eb1d5", "thisisastring"},
		{"5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8", "password"},
		{"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855", ""}, // Hash of empty-string https://www.di-mgt.com.au/sha_testvectors.html
	}

	for _, testCase := range cases {
		ReceivedBlob, err := svc.HashReverse(testCase.Hash)
		if err != nil {
			t.Errorf("Failed to calculate reverse hash for %s", testCase.Hash)
		}
		if testCase.WantedBlob != ReceivedBlob {
			t.Errorf("Failed for Hash - %s, Wanted: %s, Received: %s", testCase.Hash, testCase.WantedBlob, ReceivedBlob)
		}
	}
}

```

You can see a couple of methods with some Table driven tests. The first method `TestServiceHash` passes but the second one `TestServiceReverseHash` fails as expected. our goal will now be to fix this method so that these tests pass. Hence, TDD.

This is a good stopping point and we'll pick-up in the following post for more! Find the entire code tree at this point here: [GITHUB - TREE](https://github.com/tchaudhry91/rainbow/tree/a926df8d502dead516929efbecb72f624649aa96)

---
