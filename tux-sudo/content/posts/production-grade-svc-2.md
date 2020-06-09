---
title: "Production Grade Web Services with Go - The Store"
date: 2020-06-08T12:16:15+05:30
categories: ["development", "go", "webservice", "tutorial"]
---

Note : This is part two of a series of posts describing how to write "Production Grade Webservice in Go". Here's [Part - 1, The Service ](/posts/production-grade-svc-1/) if you haven't read it.

The previous post ended with a defined structured for our service and some basic testing. It was, however, lacking what is a very important component for most webservices, a **datastore**. If you've used frameworks to write services in the past, you're probably familiar with abstractions like Hibernate/DjangoORM etc. While Go has the option of working with similar alternatives ([GORM](https://gorm.io/), [Pop](https://github.com/gobuffalo/pop)), I generally find building a simple custom abstraction over the database to be better. Unless you have a lot CRUD like APIs with plenty of models, direct is, in my opinion, better. See this [post](https://eli.thegreenplace.net/2019/to-orm-or-not-to-orm/) for more information.

This point is specifically emphasized with our RPC like service where *resources* are not at the forefront. There is effectively no resource involved and modelling this with an ORM sounds rather clunky. Just think about it in terms of behaviour, what does our datastore really need to provide? Hopefully, you'll find an answer similar to the interface below:

```go
// Store is an interface defining the operations required from a Rainbow Store
type Store interface {
	Put(blob string, hash string) error
	Get(hash string) (blob string, err error)
}
```

This resembles a key-value store, but we're not obligated to use one. We can use any concrete implementation of the store. It could be any of Redis, Postgres, Mongo, Cosmos etc. As long as you build a wrapper that provides the given behaviour, the underlying database does not matter. I do however recommend, that proper research be done while choosing the database. While, you *can* technically switch out implementations, in more complicated services it's usually a mess with migrations and what not.

Nevertheless, let's start with the simplest possible datastore, an in-memory map.

```go
// InMemStore implements an in-memory map that can function as a Rainbow Store
type InMemStore struct {
	db map[string]string
}

// NewInMemStore instantiates a fresh InMemory Rainbow store
func NewInMemStore() *InMemStore {
	return &InMemStore{db: make(map[string]string)}
}

// Put stores the value of the blob and associates it with the hash as the key
func (store *InMemStore) Put(blob string, hash string) error {
	store.db[hash] = blob
	return nil
}

// Get retrieves the value from the database. Errors if the value is not found.
func (store *InMemStore) Get(hash string) (blob string, err error) {
	if blob, ok := store.db[hash]; ok {
		return blob, nil
	}
	return "", fmt.Errorf("Not Found")
}
```

Let's also write some basic tests for it: 

```go
func TestInMemStore(t *testing.T) {
	store := service.NewInMemStore()

	type TestCase struct {
		Blob string
		Hash string
	}

	cases := []TestCase{
		{"thisisastring", "572642d5581b8b466da59e87bf267ceb7b2afd880b59ed7573edff4d980eb1d5"},
		{"password", "5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8"},
		{"", "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"}, // Hash of empty-string https://www.di-mgt.com.au/sha_testvectors.html
	}

	// Add a few values
	for _, c := range cases {
		err := store.Put(c.Blob, c.Hash)
		if err != nil {
			t.Errorf("Error returned while added entry: %s", c.Blob)
		}
	}

	// Retrieve the values
	for _, c := range cases {
		blob, err := store.Get(c.Hash)
		if err != nil {
			t.Errorf("Error while fetching expected entry: %s", c.Hash)
		}
		if blob != c.Blob {
			t.Errorf("Mis-matched Blob for %s, Want=%s, Have=%s", c.Hash, c.Blob, blob)
		}
	}

	// Check for non existent value
	_, err := store.Get("123123")
	if err == nil {
		t.Errorf("Failed to error for a non-existent value")
	}
}
```

The tests pass! Great, now let's fix our old test that's still failing. For that we need to use a Store in our service. Back in `rainbow.go` let's add the dependency in our Service.

```go
// SHA256RainbowService is a SHA256 implementation for the RainbowService
type SHA256RainbowService struct {
	store Store
}

// NewSHA256RainbowService instantiates a SHA256RainbowService
func NewSHA256RainbowService(store Store) *SHA256RainbowService {
	return &SHA256RainbowService{store}
}
```

We added the store as a dependency to the concrete struct type. Also notice that the dependency is of type `Store` (the interface) not any particular implementation of the store.

With the store available in our service, let's re-think the methods in our service.
The `hash` method was already operational, however, it was not storing the hash as promised. The `hashReverse` method was unimplemented. Both of these can now be fleshed out now that we have a store!

```go
// Hash returns the SHA256 sum for the given string
func (svc *SHA256RainbowService) Hash(blob string) (hashed string, err error) {
	sumA := sha256.Sum256([]byte(blob))
	hashed = hex.EncodeToString(sumA[:])
	err = svc.store.Put(blob, hashed)
	return
}

// HashReverse looks up the original string for the given hash
func (svc *SHA256RainbowService) HashReverse(hashed string) (blob string, err error) {
	return svc.store.Get(hashed)
}
```

You'll notice that the signature (return value) for the `Hash` method has changed! While the hash operation cannot error, the storage to the database can! And a failure in the dependency is considered a failure in the method. This means we need to change our interface to reflect the changes as well.
The `hashReverse` method already has the correct signature and can be fleshed out as such.

```go
type Hasher interface {
	Hash(blob string) (hashed string, err error)
}
```

We only needed to edit the signature for the method in the `Hasher` interface. Our Service interface will reflect this change as well.

Now, that we have all the methods fleshed, let's try our test again after adding the store to the test service and fixing the Hash signature.

```bash
go test -v ./...
=== RUN   TestServiceHash
--- PASS: TestServiceHash (0.00s)
=== RUN   TestServiceHashReverse
--- PASS: TestServiceHashReverse (0.00s)
=== RUN   TestInMemStore
--- PASS: TestInMemStore (0.00s)
PASS
ok      github.com/tchaudhry91/rainbow/service  0.002s
```

Lovely! [Browse](https://github.com/tchaudhry91/rainbow/tree/566467618d7561ad5cb4123f589eab34aad7e225) the repository at this stage to get a better idea.

As an exercise please add the concrete implementation for the database of your choice! Another sample implementation for Redis has been added in the following [commit](https://github.com/tchaudhry91/rainbow/commit/ac3c9d9bb63d41c87f34107fcb7fc2c99095f86f).

We're two posts in and we **still** have not touched on anything that would make this a "Web" service. I promise you, we'll get there. But how cool is it to work on your business logic unencumbered by any other terminology!? More soon.