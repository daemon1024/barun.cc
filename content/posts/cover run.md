+++ 
date = "2021-10-09"
title = "Run for Cover : A Go Story"
slug = "cover run"
tags = ["Go", "Open Source", "Cloud", "Coverage", "Testing"]
type = "post"
+++

I am safe and sound, not looking for cover :P But the team at [KubeArmor](https://github.com/kubearmor/KubeArmor#kubearmor) was looking for coverage statistics for their end to end tests. I will go about how we achieved that in this post. Pretty anti dramatic considering the title uhh, but I will try to keep this post interesting :D
Before I start, let's discuss about what is code coverage and why are we running for it

## Code Coverage

Testing is one of the most critical processes of the Software Development Lifecycle (SDLC), so are the metrics associated with it. Coverage is one such metric. Coverage tells us about how much of our source code is tested. This helps us identify then untested code and add more tests to cover our bases.

## Cover Tool in Go

Go is known for it's tooling ecosystem. Go has a testing framework built in which can also generate code coverage reports. Go has a pretty creative approach to calculating coverage reports. There's no dynamic debugging but instead it rewrites the source code to enable metrics and runs the modified source code when the test actually runs. Let's take an example,

Suppose we have this package foo with the following code,

```go
package foo

func bar(a, b int) int {
	if a < b {
		return a
	}
	return b
}
```

And we add a test for that,

```go
package foo

import "testing"

func TestBar(t *testing.T) {
	t.Run("case 1", func(t *testing.T) {
		ans := bar(3, 2)
		if ans != 2 {
			t.Errorf("got %d, want %d", ans, 2)
		}
	})
}
```

Let's run our tests and generate the coverage report.

```bash
❯❯❯  go test -coverprofile=report .
PASS
coverage: 66.7% of statements
ok      run.for.cover/blog      0.001s

❯❯❯  go tool cover -html=report # HTML representation of coverage report
```

Here's the HTML representation of our report,

![https://user-images.githubusercontent.com/47106543/136659176-5accfe26-35f3-4125-a562-52546f34fba3.png](https://user-images.githubusercontent.com/47106543/136659176-5accfe26-35f3-4125-a562-52546f34fba3.png)

But how does the tool know which exact lines weren't covered. As I mentioned earlier, the source code is rewritten to add metric details. Let's see how exactly our `bar` function would have been rewritten before testing,

```go
func bar(a, b int) int {
	GoCover.Count[0] = 1
	if a < b {
		GoCover.Count[2] = 1
		return a
	}
	GoCover.Count[1] = 1
	return b
}
```

Each executable statement in the source is annotated with an assignment statement. So whenever we execute the program the assignment statement runs, when the test completes all the set counters are collected and reported to us.

Pretty clever right 😼 The official blog post [The cover story](https://go.dev/blog/cover) covers much more details about how it came into being, working, performance consideration etc. Do give it a read.

## But...

The go testing framework is mostly limited to unit tests. What if you want to test out the behavioral flow of program and wanted these metrics. What if we wanted to calculate coverage against our external test suite.

We had our own Automated System Testing Framework too and we were looking to solve the above mentioned problem.

There was no *simple* way to get these metrics against an external set of tests. But there's definitely way, a bit *hacky* but we can solve this for sure 🌚

## There's always a way

When we are executing external tests, essentially we our testing our code against the `main()` function. So we can create dummy test for our main function.

Something like

```go
package main

import "testing"

// TestMain - test to drive external testing coverage
func TestMain(t *testing.T) {
	main()
}
```

Now if we run this test, we are running our entire program as is. Now the entire main package will have those metrics added when the test is compiled and run. For ease of use, we can output a binary for the compiled test.

 Here's how we exactly generate the testable binary,

```bash
go test -coverpkg=./... -c . -o testable-binary

# -c tells to compile the test but not run it
# -o names the compiled test binary
# run `go help test` for more info
```

> Note: By default go only processes a single package, in our case `package main`. To cover all the packages, we used `-coverpkg` flag to handle all the packages present in current and sub directories. What I mean by proccessing a single package is the final coverage statistics will only have metrics from main(or the specified) package but other than that everything will function as intended. 

If you run `help` against `testable-binary` you can checkout various test flags you can set.

The one for our concern is `test.coverprofile`

## Finally

We use the binary however you would have used your normally built binary with additional test flags. When the process terminates you will have your coverage report right there.

```bash
testable-binary -test.coverprofile=system.test.out
```

In our CI pipeline, we collect all our coverage reports ( both from unit and system tests ) using [gover](https://github.com/sozorogami/gover)
and send it to [Codecov](https://about.codecov.io/) for an interactive detailed report.

That sums up our entire *Run for Cover.* Thanks for reading! Hope you found this interesting 😄. If you have any suggestions/thoughts/questions or just wanna say hi, my contact details are [here](/contact) ✌️ 

References:

- [https://go.dev/blog/cover](https://go.dev/blog/cover)
- [https://blog.cloudflare.com/go-coverage-with-external-tests/](https://blog.cloudflare.com/go-coverage-with-external-tests/)