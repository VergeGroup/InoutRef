# InoutRef

This is not an official library currently. It's for introduction how Verge detects the `commit` contains if any modifications with passing the reference of the `State`.

Using here in Verge
https://github.com/VergeGroup/Verge/blob/319c25523612baa27e016c4041c1497b68fb5622/Sources/Verge/Store/Store.swift#L175

## Motivation

### A reason why we use `inout` is to get better performance.

Verge updates its state that using `commit` with passing the reference of the State.
The reason why we use the reference is to avoid copying values.

```swift
struct MyState {
  var name = ""
}

func modify(_ value: inout MyState) {
  value.name = "muukii"
}

var state = MyState()
modify(&state)

// what properties did `modify` change?
// we can't get this.
```

Swift's struct is quite fast to copy. However, it might be serious performance issues when that structure contains a lot of properties which means it's a large struct.

As you know, Verge is a state management library, the structure of the State would be getting large.
To support that case, Verge is always using a performant way of modifying the state.

### Verge wanted to know how modified the state inside commit closure

Verge's commit is not cheap, it's often been quite expensive. Because a commit causes all of the subscribers to read the modified state.

So previously we had should care about the number of `commit` call.
Take a look example code below.

```swift

commit { state in
  if state.somethingFlag {
    state.text = "Hi"
  }
}

// how the state did change?
// state.text might be changed or not. It depends if-statement inside.
// If we could commit's modifications are empty, we stop commit.
```

In this case, `commit` does nothing updates if `state.somethingFlag` is false. 
Still, commit causes dispatching update events to all of the subscribers.

It' should be like this, Doing commit only needed to modify.

```swift
if state.somethingFlag {
  commit { state in
    state.text = "Hi"
  }
}
```

However, the reading state is outside of `commit`. It might be issues when processing concurrently.
`commit` modifies the state with getting a lock, it's a critical session.
This means that if-statement was true, the commit might be no need while stepping in. or cause breaking state.

So, switching if do commit or not should be decided inside commit session to protect from concurrent programming.
But a lot of commit gains computing costs.

### InoutRef can detect any changes happend, and received modifications with NO COPY.

`InoutRef` solves this problem. It enables Verge to detect that has any changes or not.  
Using that information, Verge can skip finalization of commit which means dispatching update events to all of subscribers.

Even though `InoutRef` is a wrapper structure, it receives the pointer of the wrapped value.  
So using `InoutRef` does not happen copy.

## Sample

```swift

struct State {

  var name: String = ""
  var count: Int = 0
}

func modify(ref: InoutRef<State>) {
  ref.count = 1
}

var state = State()

withUnsafeMutablePointer(to: &state) { (pointer) -> Void in
  let ref = InoutRef<State>(pointer)
  modify(ref: ref)
  
  print(ref.modification!)
  
}

```

output
```
Modified:
  Swift.WritableKeyPath<__lldb_expr_11.State, Swift.Int>
```
