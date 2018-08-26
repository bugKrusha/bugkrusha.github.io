# Prepping for a talk...not for sharing.
# Make It Generic

## Introduction
Can I have a `T`? If you are perplexed, that would be reasonable. In the real world, it is virtually impossible for you to respond to this without more context or specialization. Do I need a medium, do I need it hot, or should it be uppercase? However, in software engineering and Swift specifically, generally when you see the uppercase `T`, we are employing one of the most powerful paradigms in modern software engineering, _Generics_.

```Swift 
struct Resource<T> {
    let attribute: T
    
    ///
}
```

Generics aims to increase flexibility by expanding the possibilities of parameterization into the type system. That is, you can fill an object’s field with whatever type you desire. The `attribute` field in `Resource` above can be filled with any type. The beauty here is that generics enhances safety without _hampering_ performance. 

To different people, _Generics_ means different things. Today we will be focusing on _Parametric Polymorphism._ Sounds complicated? My friend Ian would describe this as having a million dollar name for a ten dollar concept. Parametric means “can be expressed as a parameter”, so like any of the fields in this resource object. Polymorphism? Sounds bright, not really. The prefix _poly_ means many and _morphism_ means form. So in this context, parametric polymorphism means having parameters that can be expressed in many forms.

## Abstraction Specialization Phases
Software engineering proceeds in what can be described as an _Abstraction_, _Specialization_ cycle. The abstraction phase involves identifying and classifying commonalities. In this fashion, we eliminate irrelevant details and focus on the essentials, gaining a broader understanding of the elements comprising our systems. In the end we arrive at a collection of laws that drive the *specialization* phase. During *specialization*, the general laws are used to instantiate specific cases required to make our systems work. This can lead to incredibly novel applications.

## Parameterization
Generic programming is normally manifested as some form of parameterization. Remember, this just means to be expressed in terms of parameters. Often times when you have a list of specific programs, you can remove the differences that lead to those specific programs and end up with a single unified generic program. Then, instantiating this unified program with various parameters causes the program to specialize itself and be expressed differently.

### Types
There are different types of Generics. Today, we will cover value, type and function.

#### Value: Ascii Art

```
*
**
***
****
*****
```
Early on in our careers, we learn to parameterize functions as we became frustrated with the futility of hard coding values into our programs. If we wanted to draw a triangle with ascii art, as shown here, we could start like this:

```Swift 
func drawAsciiArt() {
    print("*")
    print("**")
    print("***")
    print("****")
    print("*****")
}
```
The benefits of abstracting the drawing behavior into parameters becomes immediately obvious. 

```Swift 
func drawAsciiTriangle(height: Int) {
    for row in 1...height {
        var line = ""
        for _ in 1...row {
            line.append("*")
        }
        print(line)
    }
}
```
Note that this then, can be further parameterized to get a function that takes the ascii art that should be used to draw the triangle. In the end we get a program with formal parameters, performing different but related computations based on the actual parameters passed in. Beautiful.

```Swift 
func drawAsciiTriangle(height: Int, ascii: Character) {
    for row in 1...height {
        var line = ""
        for _ in 1...row {
            line.append(ascii)
        }
        print(line)
    }
}
drawAsciiTriangle(height: 7, ascii: "😘")
```
😘
😘😘
😘😘😘
😘😘😘😘
😘😘😘😘😘
😘😘😘😘😘😘
😘😘😘😘😘😘😘


#### Type
A stack is a common data structure. They can be backed by an array and have two important functions, `push` and `pop`, where `push` adds and element and `pop` removes the last element that was added.  They are described as first in last out (FILO). That is, the first element added by `push` will be the last removed by `pop`. If we wanted a stack of emojis, we could make something like this.

```Swift 
struct EmojiStack {
    var list = [Character]()
    
    var count: Int {
        return list.count
    }
    
    /// Add a new object to the stack.
    ///
    /// - Parameter value: Object to be added.
    mutating func push(value: Character) {
        list.append(value)
    }
    
    /// Remove and return last object added. Nil will be returned if
    /// if the list is empty.
    ///
    /// - Returns: Last objected added if there is one.
    mutating func pop() -> Character? {
        return list.popLast()
    }
}
```
Then, we realize we need a stack for numbers. We can then make something like this. 

```Swift 
struct IntStack {
    var list = [Int]()
    
    var count: Int {
        return list.count
    }
    
    /// Add a new object to the stack.
    ///
    /// - Parameter value: Object to be added.
    mutating func push(value: Int) {
        list.append(value)
    }
    
    /// Remove and return last object added. Nil will be returned if
    /// if the list is empty.
    ///
    /// - Returns: Last objected added if there is one.
    mutating func pop() -> Int? {
        return list.popLast()
    }
}
```

Not only is it tedious to repeat similar definitions like this, it is unsafe and difficult to maintain. The definition of the backing  `data` array is essentially identical, as are the `push` and `pop` functions. The only difference is the type in the backing array and the name of the `Stack`. Abstracting away the hard coded type leads to the creation of a single unifying polymorphic type, parameterized by another type, the type that will be the backing `data` array. At work here is the fact that the instantiated behavior is independent of the type passed in. We just want to be able to add, remove and store elements, regardless of the element's identity or contents. 

```Swift 
struct Stack<Element> {
    var list = [Element]()
    
    var count: Int {
        return list.count
    }
    
    /// Add a new object to the stack.
    ///
    /// - Parameter value: Object to be added.
    mutating func push(value: Element) {
        list.append(value)
    }
    
    /// Remove and return last object added. Nil will be returned if
    /// if the list is empty.
    ///
    /// - Returns: Last objected added if there is one.
    mutating func pop() -> Element? {
        return list.popLast()
    }
}

let intStack = Stack<Int>()
intStack.push(value: 23)

let emojiStack = Stack<Character>()
emojiStack.push(value: "🤣")
```
Now we can create a stack of any arbitrary type without tediously repeating definitions. Note that this is similar to generics by value, but at the _type_ level. We are not hardcoding the type, making our programs incredibly flexible and powerful.

#### Function
One of my favorite form of *Genericity* is by function. In Swift functions are first class citizens (typically a type that can be passed as a parameter, returned from a function or be assigned to a variable). As a result, Swift has robust support for higher order functions, where functions can be parameterized by other functions. That is, you can pass a function to another function. If you had a list of strings that you wanted to convert to lowercase for standardization, you may write something like this:

```swift
func makeLowerCase(list: [String]) -> [String] {
    var lowerCaseList: [String] = []
    
    for string in list {
        lowerCaseList.append(string.lowercased())
    }
    return lowerCaseList
}
```

Then if you had a list of of numbers you wanted to classify as even or odd using booleans. This could be written as:

```swift
func evenOdd(list: [Int]) -> [Bool] {
    var evenOddBools: [Bool] = []
    
    for number in list {
        evenOddBools.append(number % 2 == 0)
    }
    return evenOddBools
}
```
Functions that follow this pattern are very common. The only difference here is the function to be applied to each element in the list. For the first case, the `lowercased` function and for the second, it is this condition, that the number is even. What is common here is traversing the list and applying that func to each element.This is called `map` and can be defined as follows.

```swift 
extension Array {
    func map<T>(_ transform: (Element) -> T) -> [T] {
        var result: [T] = []
        for x in self {
            result.append(transform(x))
        }
        return result
    }
}
```

This isn’t genericity by value where the value is just a function. The implications here are far reaching, providing incredibly powerful customization points for engineers, without extending the language. Here is how we could get result of the former examples using `map`.

##### Lowercase
```Swift
let lowercased = ["ONE", "LOVE", "mi", "bredda"].map { $0.lowercased() }
```
##### Even Odd List
```Swift 
let evenOddList = Array(1...20).map { $0 % 2 == 0 }
```
Let’s take this even further. Say we have two lists that we want to append to together. We could write that like this:

```Swift
func appendLists(strings: [String]) -> String {
    var bigList = ""
    
    for string in strings {
        bigList.append(string)
    }
    return bigList
}
```
And then later, we have a list of numbers for which we would like to obtain its sum. We could write it like this:

```Swift
func sum<T: Numeric>(numbers: [T]) -> T {
    var total: T = 0
    for number in numbers {
        total += number
    }
    return total
}
```
These programs can be unified since they both traverse the list in the same way. Stepping back, we can see that they have a common pattern of recursion. That is, given a list, recursively apply a combine operation, building up a return value by recombining the results of processing the each element in that list. In Swift, this is called `reduce` and can be defined as follows.

```Swift 
extension Array {
    func reduce<Result>(_ initialResult: Result, _ nextPartialResult: (Result, Element) -> Result) -> Result {
        var result = initialResult
        for x in self {
            result = nextPartialResult(result, x)
        }
        return result
    }
}
```

Then we could call the other functions like this:
```Swift
let total = [90, 78].reduce(0, +)
let bigList = ["ONE", "LOVE", "mi", "bredda"].reduce("", +)
```
## JSON
You might be familiar with JSON,  Javascript Object Notation a text format that is used to serialize structured data, with the primary purpose of transmitting data between a server and an application. Here is an example:

```JSON
{
    "firstName": "Jah Zie",
    "lastName": "Tini",
    "age": 35,
    "address": {
        "streetAddress": "247 My Bag",
        "city": "Seattle",
        "state": "WA",
        "postalCode": "98104"
    }
}
```
Modeling this is more or less straightforward and might look like this:
```Swift
struct Person: Codable {
    let firstName: String
    let lastName: String
    let age: Int
    let address: Address
}

struct Address: Codable {
    let street: String
    let city: String
    let state: String
    let postalCode: String
}
```
Things can get complicated very fast when modeling some JSON however and there are OPINIONS about how it should be formatted. So, there is now a JSON API that helps to standardize things. Broadly, JSON that conforms to these specifications might look like this:

```JSON
{
    "meta": {
        "type": "identifier",
        "id": "0b6a12ec-343d-4830-b029-4ed648e4c5d7",
        "resource_type": "print"
    },
    "data": {
        "type": "print",
        "id": "1c871eec-44e7-4123-b14e-3e51646f6d5c"
    }
}
```
They can have a meta tag containing non-standard meta information. They also have a data tag that is a single resource object or a resource identifier object that is used to identify an object. At Glowforge, we have a Printer that we model as a `Machine` and its prints are modeled as `PrintActivity`. So we often get socket messages about a `Machine` or a `PrintActivity` update. Those messages are sent as resource identifiers. Socket messages can be sporadic so when we get each as a resource identifier, we can determine if we need to make a call the get the actual resource. 

### Resource Identifier
From this `JSON`, we we would like to generate a resource identifier that we can then pass to some object manager that will then make a network request to get the actual resource. 

```Swift 
struct ResourceIdentifier<T: ResourceRequestable> {
    let meta: Meta
    let data: T
}
```

The power that is wielded by generics can be exponentiated when it is combined with protocols. The possibilities are virtually endless. Here the resource identifier has two fields, `meta`, containing the meta information and `data` containing the information required to generate a network request. Here we employ what can be described as _Inclusion Polymorphism_. That is, the data object can be anything, as long as that thing conforms to `ResourceRequestable`. `ResourceRequestable` requires an associated type, which will be the type of the `Resource` object that we are expecting from a network call. It also requires a conforming types have a `type` and an `id` field and an `incomingRequest` func that will return a `APIRequest`.

```Swift
protocol ResourceRequestable: Codable {
    associatedtype Payload

    var id: String { get }
    var type: ResourceType { get }
    func incomingRequest() -> APIRequest<Payload>
}
```

### Meta
```Swift 
enum ResourceType: String, Codable {
    case printActivity
    case machine
}

struct Meta: Codable {
    let type: MetaType
    let resourceType: ResourceType
    let id: String
}
```

The meta tag has a resource type that that tells us what the resource is. For us this could be `print`, `machine` and variety of others. Today we will focus on these two.

### Resource Packet
```Swift
struct ResourcePacket<T>: ResourceRequestable {
    typealias Payload = T /// The object type being identified.
    let type: ResourceType
    let id: String
    
    func incomingRequest() -> APIRequest<Payload> {
        return APIRequest(id: id)
    }
}
```
I will call the object that represents _data_ `ResourcePacket`. This conforms to `ResourceRequestable`. The associated type is purely Generic with no inclusion constraints. That is, it can be anything. 

### API
```Swift 
struct APIRequest<T> {
    let id: String
    
    ///
}

class API {
    func perform<T>(_ request: APIRequest<T>, completion: (T) -> Void) {
        /// Make network request
    }
}
```

Now, we can make an `api` like this. Here we have a `perform` function. It too is `generic`, being able to make a network call for any resource, and handle the results of that call with a completion handler that also accepts any resource :grimacing: This is incredible.

### Object Manager
```Swift
class Manager {
    func updateReceived<T>(identifier: ResourceIdentifier<T>, newData: @escaping(T.Payload) -> Void) {
        let request = identifier.data.incomingRequest()
        api.perform(request) { incoming in
            newData(incoming)
        }
    }
}
```

Now, in your object manager, you can have a function `updateReceived` with this signature. It is generic too but the those types must be of type `ResourceRequestable`. It will get the `APIRequest` from the identifier and perform the network request.

### Conclusion
When building software systems with generics, we can decompose components in a manner that makes little to no assumptions about the other components, providing for maximum flexibility in composition. Every data type has a set of requirements. Generics emerged out of the insight that highly reusable software components must be built with a minimum set of such requirements. However, we can’t fully leverage generic programming by just identifying a set of minimal requirements for a single data type or algorithm. We must identify a set of axioms for a collection of similar types. This way we can build remarkably powerful systems with reusable components that enhance the safety and reliability of said systems. Combined with protocols, there is nothing you can’t build.


