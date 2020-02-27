## Pattern Matching in Rust

Pattern matching is one of those features that are still lacking in C/C++. Although it's not a new concept, it is   rendered a new feature in a context where pattern matching is supported in Rust as oppose to the almost absence in C/C++.

In this summary, we'll take a look at pattern matching in Rust and why it is essential.

### The Syntactic Sugar

At first glance, pattern matching looks like syntactic sugar for `switch` statement, a simplistic example of calculating Fibonacci in Rust:

```rust
fn Fib(n: i32) -> i32 {
    match n {
        1 => 1,
        2 => 2,
        _ => Fib(n - 1) + Fib(n - 2),
    }
}
```

It looks quite similar to the `switch` replica:

```c++
int Fib(int n) {
    switch (n) {
    case 1: return 1;
    case 2: return 2;
    default: return Fib(n - 1) + Fib(n - 2);
    }
}
```

However, pattern matching is capable of multiple matches with supports for decomposing and binding:

```rust
fn Tell(point: &(i32, i32)) -> String {
    (0, 0) => String::from("the point is the origin"),
    (0, _) => String::from("the point is on x-axis"),
    (_, 0) => String::from("the point is on y-axis"),
    (x, y) => format!("arbitrary point ({},{})", x, y),
}
```

Obviously, the `switch` statement won't be helpful in this case.  When the control flow and data structure get more complicated, pattern matching will become more of a necessity than a quality of life change, as we'll demonstrate with a somewhat practical example in the next section.

### The Qualitative Change

In mobile network systems, session management services communicate with each other using messages. The specific business procedure that a message invokes depends on its contents. Since the message often contains a lot of parameters, the dispatching logic can get exceptionally complex at times.

Take the update request sent from AMF to SMF as an example. The update data, defined as `SMContextUpdateData` in the specification, can represent a dozen of business procedures with different combinations of nearly 40 fields in the data.

To get a taste of what it looks like without pattern matching, we can implement the dispatching in C++ using  if-else statements:

```C++
struct SMContextUpdateData {
	std::string servingNfId;
	int upCnxState;
	int hoState;
	bool toBeSwitched;
	bool failedToBeSwitched;
	std::optional<int> n1SmMsg;
	std::optional<int> n2SmInfo;
	int n2SmInfoType;
	bool dataForwarding;
	std::optional<int> n9ForwardingTunnel;
	std::vector<int> revokeEbiList;
	bool release;
};

auto constexpr ERROR = "error";

std::string DispatchWithN2(const SMContextUpdateData& message) {
	switch (message.n2SmInfoType) {
	case 1: {
		return "n2procedure_1";
	}
	case 2: {
		if (message.toBeSwitched) {
			return "n2procedure_2";
		} else {
			return ERROR;
		}
	}
	case 3: {
		if (message.hoState == 1) {
			return "n2procedure_3";
		} else {
			return ERROR;
		}
	}
	case 4: {
		return "n2procedure_4";
	}
	case 5: {
		if (message.failedToBeSwitched) {
			return "n2procedure_5";
		} else {
			return ERROR;
		}
	}
	default: return ERROR;
	}
}

std::string DispatchWithoutN1N2(const SMContextUpdateData& message) {
	if (message.release) {
		return "releaseprocedure";
	}
	if (message.hoState == 1 && message.dataForwarding) {
		return "hoprocedure";
	}
	if (message.dataForwarding) {
		return "fowardingprocedure";
	}
	if (message.n9ForwardingTunnel) {
		return "n9forwardingprocedure";
	}
	if (message.upCnxState == 1) {
		return "serviceprocedure";
	} else if (message.upCnxState == 2) {
		return "deactivateprocedure";
	}
	if (!message.servingNfId.empty()) {
		return "interprocedure";
	}
	if (!message.revokeEbiList.empty()) {
		return "revokeprocedure";
	}
	return ERROR;
}

std::string Dispatch(const SMContextUpdateData& message) {
	if (message.n1SmMsg) {
		return "n1procedure";
	} else if (message.n2SmInfo) {
		return DispatchWithN2(message);
	}
	return DispatchWithoutN1N2(message);
}
```

For the sake of simplicity, only a subset of the data fields are selected and complex data structs are replaced with basic types like `int` and `std::string`.

`Dispatch` simulates a dispatching process where the execution path depends on the contents of the received message. As we can see when branches get longer and deeper, it's becoming quite lengthy. The same control flow can be represented a lot more succinctly with pattern matching in Rust:

```rust
struct SMContextUpdateData {
    servingNfId: String,
    upCnxState: i32,
    hoState: i32,
    toBeSwitched: bool,
    failedToBeSwitched: bool,
    n1SmMsg: Option<i32>,
    n2SmInfo: Option<i32>,
    n2SmInfoType: i32,
    dataForwarding: bool,
    n9ForwardingTunnel: Option<i32>,
    revokeEbiList: Vec<i32>,
    release: bool,
}

fn Dispatch(message: &SMContextUpdateData) -> &str {
    match message {
        SMContextUpdateData { n1SmMsg: Some(_), .. } => "n1procedure",
        SMContextUpdateData { n2SmInfo: Some(_), n2SmInfoType: 1, .. } => "n2procedure_1",
        SMContextUpdateData { n2SmInfo: Some(_), n2SmInfoType: 2, toBeSwitched: true, .. } => "n2procedure_2",
        SMContextUpdateData { n2SmInfo: Some(_), n2SmInfoType: 3, hoState: 1, .. } => "n2procedure_3",
        SMContextUpdateData { n2SmInfo: Some(_), n2SmInfoType: 4, .. } => "n2procedure_4",
        SMContextUpdateData { n2SmInfo: Some(_), n2SmInfoType: 5, failedToBeSwitched: true, .. } => "n2procedure_5",
        SMContextUpdateData { release: true, .. } => "releaseprocedure",
        SMContextUpdateData { hoState: 1, dataForwarding: true, .. } => "hoprocedure",
        SMContextUpdateData { dataForwarding: true, .. } => "fowardingprocedure",
        SMContextUpdateData { n9ForwardingTunnel: Some(_), .. } => "n9forwardingprocedure",
        SMContextUpdateData { upCnxState: 1, .. } => "serviceprocedure",
        SMContextUpdateData { upCnxState: 2, .. } => "deactivateprocedure",
        SMContextUpdateData { servingNfId: id, .. } if !id.is_empty() => "interprocedure",
        SMContextUpdateData { revokeEbiList: list, .. } if !list.is_empty() => "revokeprocedure",
        _ => "error",
    }
}
```

Immediately we can see that the code is quite a bit shorter. Maybe that's due to the compress of the curly brackets, but the absence of the nested `if` statements significantly improves the readability. Moreover, because the error handling of all the branches is identical in this case, there is no need to define an error handling branch for every layer of the nesting. To add the icing on the cake, pattern matching can check unreachable and unhandled cases at compile-time, which is hard to manage when you have multiple layers of if-else statements.

We can safely say that pattern matching is not simply a glorified version of the `switch` statement, the benefits it introduces bring quantitative changes into qualitative changes as the size scales up.

### Pattern Matching and Error Handling

It's necessary to mention error handling when it comes to patterning matching in Rust. 

For starters, error code is a common practice to deal with invalid cases in C/C++:

```c++
variant<int, errc> NumstrSum(string_view str1, string_view str2) {
	int num1;
	if (auto [p, ec] = std::from_chars(str1.data(), str1.data() + str1.size(), num1);
		ec != std::errc()) {
		return ec;
	}
	int num2;
	if (auto [p, ec] = std::from_chars(str2.data(), str2.data() + str2.size(), num2);
		ec != std::errc()) {
		return ec;
	}
	return num1 + num2;
}
```

Error code is also quite common (actually preferred) in Rust. A typical implementation of `NumstrSum` using pattern matching in Rust would be:

```rust
fn NumstrSumExpanded(str1: &str, str2: &str) -> Result<i32, ParseIntError> {
    let num1: i32;
    match str1.parse() {
        Ok(r) => num1 = r,
        Err(e) => return Err(e),
    }
    let num2: i32;
    match str2.parse() {
        Ok(r) => num2 = r,
        Err(e) => return Err(e),
    }
    Ok(num1 + num2)
}
```

It's less than ideal, however, in that it is very verbose. In the world of Rust, you wouldn't normally see code written with this style (unless it's written by newbie Rustaceans such as me). Instead, what you'll see are question mark operators:

```rust
fn NumstrSum(str1: &str, str2: &str) -> Result<i32, ParseIntError> {
    let num1: i32 = str1.parse()?;
    let num2: i32 = str2.parse()?;
    Ok(num1 + num2)
}
```

This looks much nicer! Especially when there are multiple statements returning error codes, this will be significantly useful. We will unravel more on that in another summary about error handling. For now, we need to know that the question mark operator `?` is simply syntactic sugar of `match` at its core. 

The simplicity of error handling that the question mark operator `?` introduces is built on the foundation of the pattern matching in Rust, which is just another reason why pattern matching matters.

## Conclusion

Pattern matching provides a graceful solution when decomposing and processing complex data structures. It offers:

- A succinct expression of conditional matches that is less error-prone
- Exhaustive match that checks for unreachable and unhandled cases at compile-time
- Supports for simplistic error handling

Having pattern matching support adds one to the many advantages Rust has over C/C++.

