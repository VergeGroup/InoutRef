# InoutRef

This is not an official library currently. It's for introduction how Verge detects the `commit` contains if any modifications with passing the reference of the `State`.

## Motivation



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
