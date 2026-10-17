### 3.1 `lift.current_floor_ = 5;` after declaring `lift`

```
main.cpp:8:10: error: 'current_floor_' is a private member of 'Elevator'
    8 |     lift.current_floor_ = 5;
      |          ^
elevator.hpp:15:11: note: declared private here
   15 |       int current_floor_{1};
      |           ^
1 error generated.
```

**Which access rule makes this invalid?** `current_floor_` is declared in the `private` section of `Elevator`, so only member functions (and friends) of the class can access it. `main` is not a member of `Elevator`.

**Why is the assignment inside `go_to` allowed?** `go_to` is a member function of `Elevator`, and member functions have access to the private members of their class.

### 3.2 `fixed.go_to(2);` after the final print statement

```
main.cpp:26:5: error: 'this' argument to member function 'go_to' has type 'const Elevator', but function is not marked const
    26 |     fixed.go_to(2);
       |     ^~~~~
elevator.hpp:9:12: note: 'go_to' declared here
    9 |       bool go_to(int requested);
      |            ^
1 error generated.
```

**What does the trailing `const` on `current_floor` allow?** It allows the function to be called on a `const` object such as `fixed`. It is a promise that the function does not modify the object.

**Why is it unsuitable for `go_to`?** `go_to` modifies `current_floor_`, so it cannot promise to leave the object unchanged. If it were declared `const`, it could not do its job (and it would then be callable on `const` objects, which should not be moved).

### 3.3 `Elevator spare;` after declaring `lift`

```
main.cpp:8:14: error: no matching constructor for initialization of 'Elevator'
    8 |     Elevator spare;
      |              ^
elevator.hpp:7:16: note: candidate constructor not viable: requires single argument 'top_floor', but no arguments were provided
    7 |       explicit Elevator(int top_floor);
      |                ^        ~~~~~~~~~~~~~
elevator.hpp:4:7: note: candidate constructor (the implicit copy constructor) not viable: requires 1 argument, but 0 were provided
elevator.hpp:4:7: note: candidate constructor (the implicit move constructor) not viable: requires 1 argument, but 0 were provided
1 error generated.
```

**Why can `Elevator` not use an implicit default constructor?** The compiler only generates a default constructor if the class declares no constructors at all. `Elevator` declares `explicit Elevator(int top_floor)`, so the implicit default constructor is not generated.

**Why do the default member initializers `{1}` not provide the missing constructor?** They only give the data members their initial values when some constructor runs. They are not constructors themselves, and they cannot create an object on their own without a constructor call.

## 4.  Construction and destruction order
Expect the following output from running `main`:

```
construct 1
construct 2
inspect 2
destroy 2
destroy 2
destroy 2
construct 11
destroy 11
construct 12
destroy 12
construct 20
construct 21
destroy 21
destroy 20
construct 20
construct 22
end of leave_early
destroy 22
destroy 20
end of main
destroy 1
```

- `destroy 2` appears three times because three separate objects contain the ID `2`. The first is the parameter `probe` in `inspect`, created by copying `copy` when it is passed by value. The second is `copy`, created by `Probe copy = two;` using the implicit copy constructor. The third is `two`. The copy constructor does not print anything because it was not explicitly defined.
- The objects are destroyed in reverse order of their completed construction within each scope. When `inspect` returns, its parameter is destroyed. At the end of the inner block, `copy` and then `two` are destroyed.
- Each loop iteration creates a new `item`. The loop body has its own scope, so the item is destroyed at the end of each iteration. This produces separate construction and destruction messages for IDs 11 and 12.
- In `leave_early(true)`, `first` is constructed before `second`. The `return` statement skips the construction of `third` and the print statement `end of leave_early`. Both existing local objects are still destroyed: `second` first, then `first`.
- In `leave_early(false)`, `second` is skipped. Instead, `third` is constructed. The program prints `end of leave_early`, then destroys `third` and `first`.
- At the end of `main`, `one` is destroyed after `end of main` is printed.
## 5. Explanation

1. `Elevator service = lift;` creates a new object, so it uses the **copy constructor**. `lift = service;` assigns to an object that already exists, so it uses the **copy assignment operator**. Both are available because the compiler generates them implicitly (member-wise copy) when the class does not declare or delete them.
2. A constructor has no return type, so it cannot return `false` like `go_to`. A class can respond to an invalid constructor argument by **throwing an exception** (for example `std::invalid_argument`). The object is then never created, so no object with a broken invariant exists.
3. No. `private` only prevents code outside the class from changing `current_floor_`; it does not guarantee that the class itself keeps the value correct. The invariant is preserved by the constructor, which starts the elevator at floor 1, and by `go_to`, which checks `requested < 1 || requested > top_floor_` before assigning. If `go_to` assigned the requested floor first and checked afterwards, an invalid request would already have changed `current_floor_`, and returning `false` would leave the object in an invalid state.
4. Yes, an `Elevator` is still destroyed automatically when its scope ends, because the compiler generates a destructor if none is declared. A hand-written `~Elevator()` would add nothing, since the class owns no resources (just two `int` members) that need to be released.