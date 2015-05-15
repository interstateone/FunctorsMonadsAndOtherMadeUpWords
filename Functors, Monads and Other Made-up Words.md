^ Why should we care about functional languages and functional programming?
^ What does functional mean? expressions > statements, higher-order functions, immutable data types, limitations of (or no) side effects

# Functors, Monads and Other Made-Up Words

^ I recently went to NSNorth and saw a presentation by Gordon Fontenot at Thoughtbot on functors, and he did such a great job that I felt really compelled to learn more about this really alien-sounding concept. With all of the exploring and writing that people have been doing with Swift, there's been a lot of different opinions and paths that people have been taking with it. One of the more vocal groups of people that I've seen is those that have explored Swift's potential for functional programming. I haven't spent too much time with Swift yet, only a few small projects or playgrounds, and it's definitely been overwhelming at times where I've just shut my computer and walked away for a few days or weeks. I wanted to give this talk as a way to force myself to explore these idioms more and condense other's blog posts and the theory behind it.
^ I want to talk about three concepts today: functors, applicatives and monads. I'm going to relate or borrow some things from the Haskell language as well, but all of my examples and code will be in Swift. If you haven't written much Swift yourself yet, that's okay because the examples aren't really code heavy, but you should already be familiar with some of Swift's unique features like generics and its enum type.

---

## Functors
### Mappable

^ A functor is just a type with `map` (or `fmap`) defined on it. That's it! It's probably a lot easier to understand if we were to call it mappable, but the term functor will come up a lot when learning about functional programming. So what does this mean?
^ Let's square an array of numbers.

---

```swift
let numbers = [1, 2, 3, 4, 5]
var squares: [Int] = []

for number in numbers {
    squares.append(number * number)
}

// [1, 4, 9, 16, 25]
```

^ There's some things going on here that are probably going to end up being repeated throughout the rest of a program's code. The pattern of initializing a new, empty, mutable array and then iterating over the old one happens a lot when I write apps. In fact the only part of this code that's really specific to this particular use case is the `i * i` bit.
^ That mutable array also exists in the same scope as all of this other work that we're going to be doing, even though we really only need it for future work at the end, when it contains all of the squared numbers. Because it's a mutable array, there's always a chance that it gets changed because of some later statements in the same scope. This is a trivial example and it would likely be difficult to mess this up. If it were a property of an object that has references elsewhere in our program, sometimes these changes can be hard to track down.
^ Contrast this with the use of a map function. You might have already used map as a replacement for this imperative iteration. Here the only bit that we're left with that's specific to this use case is, well, the only bit that's specific to this use case!

---

```swift
let mappedSquares = map(numbers, { number in
    return number * number
})

// [1, 4, 9, 16, 25]
```

^ It's easy imagine what's going on under the hood in this example, but for the sake of the next point lets write out an implementation for this function.

---

![original](IMG_0269.jpg)

---

^ The I and O are called placeholder types in Swift and represent _any_ types. An I could be a String, or an Int, or a NSDate. By calling it I we let the compiler know that it could be any of those types of objects or values, and so we only have to write this kind of map function once. The other thing to note about this is that by using I and O throughout the implementation, we tell the compiler that we expect the same type in certain places. Let's say that O is an Int. So we expect the transform function to return a single Int. We need to then use an Int for the result array and the returned array. And the compiler will check all of this for you at compile-time instead of run-time. Also, even though we're using two different type variables I and O here, they could be the same types. So we could transform a string value to another string value with map, like if we wanted to make some strings uppercase.

```swift
func map<I, O>(array: [I], transform: (I) -> O) -> [O] {
    var output: [O] = []
    for input in array {
        output.append(transform(input))
    }
    return output
}
```

---

^ The I and O are called placeholder types in Swift and represent _any_ types. An I could be a String, or an Int, or a NSDate. By calling it I we let the compiler know that it could be any of those types of objects or values, and so we only have to write this kind of map function once. The other thing to note about this is that by using I and O throughout the implementation, we tell the compiler that we expect the same type in certain places. Let's say that O is an Int. So we expect the transform function to return a single Int. We need to then use an Int for the result array and the returned array. And the compiler will check all of this for you at compile-time instead of run-time. Also, even though we're using two different type variables I and O here, they could be the same types. So we could transform a string value to another string value with map, like if we wanted to make some strings uppercase.

```swift
func map<I, O>(array: [I], transform: (I) -> O) -> [O] {
    var output: [O] = []
    for input in array {
        output.append(transform(input))
    }
    return output
}
```

![inline original](IMG_0269.jpg)

^ There's two important changes once we use map instead of the imperative method:

^ 1. We're telling the computer what we want it to do (square each value in the array) but not how to do it (for each value apply the function to the value and put it in the new array).
^ 2. We eliminate any mutable state in _our_ code.

^ The map implementation we wrote completely encapsulates both of these concepts (iteration and mutability) for us, and makes our code safer (accidental mutation) and more reusable as a result

^ Swift's Array type already has a map implementation that we can use, and so Swift's Array type is a functor because it has a map function.

---

```swift
map(numbers, { number in number * number })

// or:

numbers.map({ number in number * number })
```

^ When we use map here, the placeholder types I and O that we declared before in the implementation are "replaced" with Int and Int, because the Swift compiler is able to infer that we're passing in `arr` (an array of Ints) and the transform function will return an Int (Int * Int == Int).

---

![bottom original](IMG_0270.jpg)

---

^ Optional is many things, most simply (and reductively) being a way for Swift to interoperate with Objective-C's nil value. Optional is a way of expressing the existence of a value as a type. For example, instead of a method returning an `NSString *`, in Swift it would return a `String?`, which reads as "optional string". This means that the author and compiler both know that the method will return *either* a String or nil. This is distinct from a method returning `String`, which enforces that the method *must* return one, and can't return nil unless something goes really wrong. In Objective-C the return type `NSString *` can't make this distinction. The ? syntax is just sugar, and so `String?` translates to `Optional<String>`

![](IMG_0280.jpg)

---

^ If SourceKit's code completion has worked for you, you might have seen it suggest a map function for Optional too. Before looking at Optional's map, let's consider an imperative way of working with Optionals.

^ Let's say we have an app for our exclusive club that requires a passcode to get in. The passcode is just the square of the number of letters in the member's name. This is a treehouse club, not a crypto club. How would we handle nil names when computing the expected passcode?

^ First, lets define a function called square to make it more clear what we're doing in each step.

```swift
func square(num: Int) -> Int { return num * num }

func calculatePasscode(input: String?) -> Int? {
    let passcode: Int?
    if let i = input {
        let number = count(i)
        passcode = square(number)
    }
    else {
        passcode = Optional.None
    }
    return passcode
}
```

^ One nicety here is that Swift 1.2 now lets us make passcode a constant because it'll only be assigned to once in this scope, where previously it would have needed to be declared with `var`.

^ This works, but that's still a lot of telling the computer exactly what we want it to do. Let's look at the function signature for Optional's map now.

---

![](IMG_0271.jpg)

---

![](IMG_0272.jpg)

---

```swift
// Array
func map<I, O>(array: [I], transform: (I) -> O) -> [O]

// Optional
func map<I, O>(optional: I?, transform: (I) -> O) -> O?
```

---

^ Let's write the implementation:

```swift
func map<I, O>(optional: I?, transform: (I) -> O) -> O? {
    if let o = optional {
        return Optional.Some(transform(o))
    }
    return Optional.None
}
```

^ So we only transform the value if there is one, otherwise we return Optional.None. Similar to how Array's map encapsulates iteration, Optional's map encapsulates checking for a contained value. One interesting thing to note is that the transform function doesn't need to deal with Optional values at all, map handles that for us.
Let's try rewriting `calculatePasscode` to use map.

---

```swift
func calculatePasscode2(input: String?) -> Int? {
    let number = map(input, count)
    let passcode = map(number, square)
    return passcode
}
```

^ To be clear, since I've written this fairly compactly, first we map `count` over the input, and then we map square over the result of that calculation. If the result of the first calculation is .None, then the second use of map will handle this for us and `square` is none the wiser. Neither `count` and `square` care about Optionals at all. In this example we're just composing two existing functions in a safe way. Let's use it:

---

```swift
func calculatePasscode2(input: String?) -> Int? {
    return map(map(input, count), square)
}
```

---

```swift
func calculatePasscode2(input: String?) -> Int? {
    return map(map(input, count), square)
}

extension Optional {
    func map<I, O>(optional: I?, transform: (I) -> O) -> O? {
        if let o = optional {
            return .Some(transform(o))
        }
        return .None
    }
}
```

---

```swift
func calculatePasscode3(input: String?) -> Int? {
    return input.map(count).map(square)
}
```

---

```swift
let name: String? = "Steve"
let secret = calculatePasscode2(name)
// 25

let emptyName: String? = .None
let secret2 = calculatePasscode2(emptyName)
// nil
```

---

^ If we look at these two map functions again, they look pretty similar:

```swift
func map<T, U>(array:   [T], transform: (T) -> U) -> [U]
func map<T, U>(optional: T?, transform: (T) -> U) ->  U?
```

---

^ In fact, we can map over other types too, like Result. Result is an enum like Optional, but in the .None case we want an associated type like an NSError, and it's often used for failable operations like fetching some data where we need more info when it fails. You might hear it referred to as an instance of the Either monad, because it can hold only one of either values, but not both.

![](IMG_0273.jpg)

---

^ In this case we're declaring Result with only one placeholder type, but it's also possible to use two. Explain what the Box class is and why it's necessary

```swift
class Box<T> {
    let unbox: T
    init(_ value: T) {
        self.unbox = value
    }
}

enum Result<T>: Printable {
    case Value(Box<T>)
    case Error(NSError)

    var description: String {
        switch self {
        case .Value(let box):
            return "\(box.unbox)"
        case .Error(let error):
            return error.description
        }
    }
}
```

^ You can probably imagine a function like `dictionaryFromJSONFile` that returned a Result<Dictionary> (this type would mean that it returned a Result containing either a Dictionary or an NSError).

^ Let's write an implementation of `map` for Result:

---

```swift
func map<I, O>(result: Result<I>, transform: (I) -> O) -> Result<O> {
    switch result {
    case .Value(let v):
        return Result.Value(Box(transform(v.unbox)))
    case .Error(let e):
        return Result.Error(e)
    }
}
```

^ Again, this is just encapsulating the switch logic for us and only transforming the .Value case. So now our map functions look like this:

---

```swift
func map<T, U>(array: [T],        transform: (T) -> U) -> [U]
func map<T, U>(optional: T?,      transform: (T) -> U) -> U?
func map<T, U>(result: Result<T>, transform: (T) -> U) -> Result<U>
```

^ So the "shape" of these functions all seem to be the same. They take a container of one kind of value, a function that transforms values but doesn't care about the container, and returns the transformed values in the same kind of container.

^ These container types are all examples of functors in Swift! You can probably think of others, like promises, which contain a value that we don't have yet but will in the future. The important thing to remember about functors is they're a type that has a defined map function like the ones we've seen.

^ Now you might be thinking: if functors have a common interface, could we write a functor protocol so we could generalize functions over all of these types? No. 😬 Or at least not yet.

---

```swift
protocol Functor<T> {
    func map<U>(transform: T -> U) -> Self<U>
}

func map<F: Functor, T, U>(x: F<T>, transform: T -> U) -> F<U> {
    return x.map(transform)
}
```

^ This was stolen from [Gordon's](https://github.com/gfontenot/talks/tree/master/Functors) talk at NSNorth 2015. It doesn't compile because Swift doesn't support generic protocols but this is probably what it would look like if it did. Notice that it's just saying a type has a map function, and there's a free function that just calls the type's map.

---

## Applicatives

### Mappable'

^ An applicative (or applicative functor) is almost the same as a functor (it also has `map`), except the transform function that is passed is itself contained in a functor too. I'm not going to say much about Applicatives, but instead just show this:

---

```swift
func map<T, U>(optional: T?, transform: (T) -> U) -> U?
func apply<T, U>(value: T?,  transform: (T -> U)?) -> U?
```

^ except that the transform function itself is inside an Optional. Applicatives, in addition to having the map function, also have a function called apply.

---

```swift
func apply<I, O>(transform: ((I) -> O)?, value: I?) -> O? {
    if let v = value, t = transform {
        return Optional.Some(t(v))
    }
    return Optional.None
}
```

---

## Monads
### Mappable 2, The Mappening

^ Let's say that we had a list of words, and we wanted a list of all of the characters in those words. Could we use map to get that?

---

```swift
let strings = ["one", "two", "three"]
let letters = strings.map({ word in Array(word) })
// [["o", "n", "e"], ["t", "w", "o"], ["t", "h", "r", "e", "e"]]
```

---

![](IMG_0274.jpg)

^ Look at map signature and see why we get a 2D array of letters

---

![](IMG_0275.jpg)

---

^ Explain that what we want is an implementation like map but that flattens everything at the end

![](IMG_0276.jpg)

---

```swift
func flatten<T>(array: [[T]]) -> [T] {
    var flatArray = [T]()
    for dimension in array {
        flatArray.extend(dimension) // appends elements in dimension to flatArray
    }
    return flatArray
}

flatten(letters)
// ["o", "n", "e", "t", "w", "o", "t", "h", "r", "e", "e"]
```

---

^ Because we wrote a generic example, here's another test with some numbers

```swift
let twod = [[1,2,3], [4,5,6], [7,8,9]]
let oned = flatten(twod)
// [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

^ Looks like what we want, right?

^ Let's look at the map function again:

---

```swift
func map<T, U>(x: [T], t: T -> U) -> [U]
```

^ But when we gave it a transform function that returned an array of letters, that's kind of like replacing the U with [U]:

```swift
func map<T, U>(x: [T], t: T -> [U]) -> [[U]]
```

---

![](IMG_0277.jpg)

---

^ And so this is going to be a little ugly to deal with if we have nested results being returned, and this brings us to monads. Monads are also functors and applicatives, so they have map and apply functions too. Monads also have a function called bind defined on them, but you might recognize the name flatMap. Swift 1.2 added a flatMap implementation for Array and Optional. What does it look like?

```swift
func flatMap<T, U>(x: [T], t: T -> [U]) -> [U]
```

^ Better! And because we already wrote flatten and map functions, it's really easy to write flatMap:

---

```swift
func flatMap<I, O>(array: [I], transform: I -> [O]) -> [O] {
    return flatten(map(array, transform))
}
```

---

![](IMG_0274.jpg)

---

```swift
func flatMap<I, O>(array: [I], transform: I -> [O]) -> [O] {
    return flatten(map(array, transform))
}
```

^ Alright, easy peasy. I mentioned a `dictionaryFromJSONFile` function when I introduced the Result type earlier. Let's make that now:

---

^ First we need a way to load a text file into a String when we know the path.

```swift
// Get this out of the way
let fileError = NSError(domain: "com.monads.lol", code: 100, userInfo: nil)
let jsonError = NSError(domain: "com.monads.lol", code: 101, userInfo: nil)

func loadTextFile(path: String) -> Result<String> {
    let fullPath = NSBundle.mainBundle().resourcePath?.stringByAppendingPathComponent(path)
    if let f = fullPath {
        var stringError: NSError?
        let contents = NSString(contentsOfFile: f, encoding: NSUTF8StringEncoding, error: &stringError)
        if let e = stringError {
            return Result.Error(e)
        }
        if let c = contents as? String {
            return Result.Value(Box(c))
        }
    }

    return .Error(fileError)
}
```

---

^ Now we want a way to create a JSON dictionary from a String.

```swift
typealias JSON = Dictionary<String, AnyObject>

func parseJSON(json: String) -> Result<JSON> {
    let data = json.dataUsingEncoding(NSUTF8StringEncoding, allowLossyConversion: false)
    if let d = data {
        var parseError: NSError?
        let dict: AnyObject? = NSJSONSerialization.JSONObjectWithData(d, options: NSJSONReadingOptions(0), error: &parseError)
        if let e = parseError {
            return Result.Error(e)
        }
        if let j = dict as? JSON {
            return Result.Value(Box(j))
        }
    }

    return .Error(jsonError)
}
```

^ Alright, two functions that take a plain value and return a Result with either a transformed value or an error. If you squint, these two functions actually look almost exactly the same, and are probably very similar to how we could wrap many other failable Cocoa APIs. Let me help you with that:

---

![inline left original](Screenshot 2015-05-06 10.50.39.png)
![inline left original](Screenshot 2015-05-06 10.50.45.png)

^ So now lets try using these two functions to load a JSON file and get a dictionary out of it:

---

```swift
let path = "some.json"
let contents: Result<String> = loadTextFile(path)
switch contents {
case .Value(let box):
    let json = parseJSON(box.unbox)
    switch json {
    case .Value(let box2):
        println(box2.unbox)
    case .Error(let error2):
        println(error2)
    }
case .Error(let error):
    println(error)
}

// "[baz: 1234567890, bloop: 0, foo: bar]"
```

^ Nested switch statements, yum.
^ We've written some functions that maybe we can use here. Before we wrote a map function for Result, and it looked like this:

--- 

```swift
func map<I, O>(result: Result<I>, transform: (I) -> O) -> Result<O> {
    switch result {
    case .Value(let v):
        return Result.Value(Box(transform(v.unbox)))
    case .Error(let e):
        return Result.Error(e)
    }
}
```

^ This really nicely encapsulated the switch statement used to decide how to operate on a Result. Notice that the transform function returns a plain value. We can't pass in something like loadTextFile or parseJSON because they return Results. The flatMap that we wrote for Array, though, has a transform function that return an array instead of a plain value. Let's compare the signatures:

---

```swift
func map<T, U>(x: [T], t: T -> U) -> [U]
func flatMap<T, U>(x: [T], t: T -> [U]) -> [U]
```

^ flatMap might let us work with this compilcated switching logic in a cleaner way. I'll note here that sometimes when I think of map or flatMap, I often tie it to arrays of values instead of just one value (or none!), and so it can take a bit of mental work to decouple that and realize how map and flatMap just apply functions to a container. It just so happens that one of the containers it works with is an array of many values.

---

![](IMG_0284.jpg)

---

![](IMG_0285.jpg)

^ We don't have a flatMap for result yet, but we do have map. Lets also make a flatten for Result:

---

```swift
func flatten<T>(result: Result<Result<T>>) -> Result<T> {
    switch result {
    case .Value(let box):
        return box.unbox
    case .Error(let error):
        return Result.Error(error)
    }
}
```

![inline fill](IMG_0284.jpg)![inline fill](IMG_0285.jpg)

^ Based on this we can really easily make a flatMap for Result, just like it was really easy to do so for Array:

---

```swift
func flatMap<I, O>(result: Result<I>, transform: (I) -> Result<O>) -> Result<O> {
    return flatten(map(result, transform))
}
```

---

^ Let the magic wash over you

```swift
let json = flatMap(loadTextFile("some.json"), parseJSON)
// "[baz: 1234567890, bloop: 0, foo: bar]"
```

^ This works great, but trying to read this leads to some jumping around to figure out the order of operations. Bind and flatMap also have the >>= operator in Haskell, which we'll define for an easier to read sequence in this example. This will allow us to get a little fancy again and write out a sequence of operations from left to right.

---

```swift
infix operator >>== { associativity left precedence 150 }
func >>==<T, U>(x: Result<T>, t: T -> Result<U>) -> Result<U> {
    return flatMap(x, t)
}
```

---

```swift
let json2 = loadTextFile("some.json") >>== parseJSON
// "[baz: 1234567890, bloop: 0, foo: bar]"
```

^ Alright, this is getting better. It's almost like Unix's pipe operator. We could go one step further and wrap our known path in a Result type, which might be the case if we were capturing it from user input or the result of a web request.

---

```swift
let path3 = Result.Value(Box("some.json"))
let json3 = path3 >>== loadTextFile >>== parseJSON
// "[baz: 1234567890, bloop: 0, foo: bar]"

let path4 = Result.Value(Box("none.json"))
let json4 = path4 >>== loadTextFile >>== parseJSON
// NSError("The operation couldn’t be completed. No such file or directory")
```

^ Now we've got some code that barely looks like Swift anymore. Where'd we end up?

---

```swift
path2 >>== loadTextFile >>== parseJSON
```

^ Remember, this example is about monads. For a type to be a monad, it just needs to have a flatMap function defined on it.

---

^ This kind of looks like normal map, doesn't it?

```swift
func flatMap<T: Result, U, V>(T<U>, (U) -> T<V>) -> T<V>
func map<T: Result, U, V>(T<U>, (U) -> V)    -> T<V>
```

^ The only difference is what the transform function returns. Where the map transform returns a plain value, flatMap's returns another monad. With our concrete example about trying to load a JSON file into a value we saw how monads make it easy to chain together functions that don't just return a plain value.

---

## Functors, Applicative, Monads

^ Note to self: probably bring these in one at a time and briefly recap each

```swift
    protocol Functor<T>
    protocol Applicative<T>: Functor
    protocol Monad<T>: Applicative

        func map<F: Functor,     T, U>(x: F<T>, transform: (T) -> U) ->    F<U> // mappable
      func apply<A: Applicative, T, U>(x: A<T>, transform: ((T) -> U)?) -> A<U> // applicable
    func flatMap<M: Monad,       T, U>(x: M<T>, transform: (T) -> U?) ->   M<U> // chainable
```

^ This isn't compilable Swift, but you'll recognize this syntax and see how these relate.

---

# Further Reading

- [How I Learned to Stop Worrying and Love the Functor](https://github.com/gfontenot/talks/blob/master/Functors/Functors.md)
- [Functors, Applicatives, And Monads In Pictures](http://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html)
- [Learn You A Haskell](http://learnyouahaskell.com)
- [Functor and Monad in Swift](http://www.javiersoto.me/post/106875422394)
- [Flattenin' Your Mappenin'](http://robnapier.net/flatmap)
- [Deriving Map](http://dscoder.com/DerivingMap/)
- [Railway Oriented Programming](http://www.slideshare.net/ScottWlaschin/railway-oriented-programming)