---
title: "Azure SDK Go"
date: 2020-05-18T11:06:04Z
draft: true
tags: ["azure", "development", "go", "cloud"]
categories: ["Development"]
---

So, I've recently had to work with the [Azure-SDK](https://github.com/Azure/azure-sdk-for-go) for a few small tasks. I've worked with AWS and GCP SDKs before, and while they have their problems it wasn't too hard to figure them out. Azure wasn't quite the same.

Now, I am new to Azure, but I did not expect to spend 30 minutes to get a simple VM listing to work. The [Authentication](https://github.com/Azure/azure-sdk-for-go#authentication) docs seemed fairly straight forward. I just need my credentials now. Okay, google, tell me how to do that. I read the official docs and they did not seem very easy to navigate. Eventually, I figured it out. Here's the gist of it for anyone still wondering:

- Navigate to `App Registrations` in the Portal and create a new one.
- Find the newly created "App" and look for `Certificates and Secrets` in the left sidebar.
- You can create a `+ New Client Secret` from here and that should give you what you need to get started.

Hang on though, what about RBAC? Yes, so the roles need to be managed from a totally different place. The "App" is essentially considered a User. So here's what you need to assign a proper role:

- Navigate to the `Subscriptions` panel on the portal and select the subscription you want to work with.
- Again, on the left sidebar you should see `Access Control (IAM)`. Here you can create a new Role or use a Built-In and assign it to the "App" in the `Role Assignments` tab.

Fairly more complex than anything I've encountered earlier. Perhaps I'll appreciate the benefit for the workflow as I work more with it.

The rest was fairly straightforward, except the GoDoc. Despite the [GoDoc](https://godoc.org/github.com/Azure/azure-sdk-for-go) being more polluted than my eyes could handle, I stuck with it and eventually figured out the basics. Just `ctrl + f` your way through it. Here's a small example I used to list the VMs in our account:

```go
package main
import (
	"context"
	"fmt"
	"time"
	"github.com/Azure/azure-sdk-for-go/profiles/2019-03-01/compute/mgmt/compute"
	"github.com/Azure/go-autorest/autorest/azure/auth"
)
const subscriptionID = "find-your-own"

func main() {
	authorizer, err := auth.NewAuthorizerFromEnvironment()
	if err != nil {
		panic(err)
	}
	vmClient := compute.NewVirtualMachinesClient(subscriptionID)
	vmClient.Authorizer = authorizer
	ctx, cancel := context.WithTimeout(context.Background(), 100*time.Second)
	defer cancel()
	results, err := vmClient.ListAllComplete(ctx)
	if err != nil {
		panic(err)
	}
	for results.NotDone() {
		vm := results.Value()
		fmt.Println(*vm.ID)
		if err := results.Next(); err != nil {
			panic(err)
		}
	}
}
```

Now, while this works, it wasn't before I ran into another problem with version mismatches. I hadn't defined version in my `go.mod` and that seemed to trip things. So, go ahead declare the versions manually. It's something I'd normally do anyway, but not while testing things out. After some rather unhelpful compilation failures and dives into GitHub issues, it was fixed by declaring the dependency explicitly.
