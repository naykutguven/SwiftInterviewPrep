---
tags:
  - swift
  - ios
  - interview-prep
  - type-system
  - equatable
  - hashable
  - collections
---
## 0. Rubric snapshot

**Rubric expectation**

Understand equality/hash invariants and how they affect dictionaries and sets. B8 specifically focuses on collection correctness, mutable hashed state, and why hash values are not persistent identifiers.

**Caveats**

Mutable hashed state is a correctness bug. Hash values are implementation/runtime artifacts, not stable IDs. Apple’s `Hashable` documentation says the components fed into `Hasher` must be the same essential components used by `==`, and equal instances must feed the same values in the same order. ([Apple Developer](https://developer.apple.com/documentation/Swift/Hashable?utm_source=chatgpt.com "Hashable | Apple Developer Documentation"))

**You should be able to answer**

- What invariant must always hold between `==` and `hash(into:)`?
- Why is storing mutable fields in a `Hashable` key dangerous?

**You should be able to do**

- Debug a bug where a value disappears from a `Set` after a property mutation.
- Predict the behavior of the code probe.

---

## 1. Core mental model

`Equatable` defines whether two values mean “the same thing” for a domain. `Hashable` makes that equality usable by hash-based collections like `Set` and `Dictionary`.

A hash table does not store elements by scanning every value every time. It computes a hash, uses that hash to find a bucket, and then uses equality to distinguish actual matches from collisions. That means equality and hashing are not independent features. They are one semantic contract expressed in two operations.

The key rule:

```text
If a == b, then a and b must hash from the same essential components.
```

The reverse is not required:

```text
Same hash does not imply equality.
```

Hash collisions are allowed. Incorrect equality/hash relationships are not.

The dangerous case is mutable hashed state. If a value is inserted into a `Set` or used as a `Dictionary` key, then a field that participates in equality or hashing changes, the collection’s internal hash-table placement becomes stale. The collection is not notified. It does not automatically rehash the element.

This is especially easy to mess up with classes:

```swift
final class User: Hashable {
    let id: Int
    var name: String
}
```

A `Set<User>` stores references to `User` objects. Mutating `user.name` mutates the object already inside the set. If `name` participates in `==` or `hash(into:)`, the set’s invariants are broken.

The key idea:

```text
Hashable identity must be stable for as long as the value is used as a Set element or Dictionary key.
```

Swift guarantees hash-based collections work correctly when your `Hashable` conformance obeys the contract. Swift does not guarantee recovery if you mutate equality/hash-relevant state behind the collection’s back.

---

## 2. Essential mechanics

### Equality is semantic, not structural by default

`Equatable` should encode the equality that your domain actually needs.

For a value object:

```swift
struct Money: Equatable {
    let amount: Decimal
    let currencyCode: String
}

// Same amount and currency means same value.
Money(amount: 10, currencyCode: "EUR") == Money(amount: 10, currencyCode: "EUR")
```

For an entity:

```swift
struct User: Equatable {
    let id: UserID
    var displayName: String

    static func == (lhs: User, rhs: User) -> Bool {
        lhs.id == rhs.id
    }
}
```

For entities, changing display data usually should not change identity.

### `Hashable` must match `Equatable`

Bad:

```swift
struct User: Hashable {
    let id: Int
    var name: String

    static func == (lhs: User, rhs: User) -> Bool {
        lhs.id == rhs.id
    }

    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
        hasher.combine(name) // Wrong: name is not part of equality.
    }
}
```

Better:

```swift
struct User: Hashable {
    let id: Int
    var name: String

    static func == (lhs: User, rhs: User) -> Bool {
        lhs.id == rhs.id
    }

    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }
}
```

If equality says two users with the same `id` are equal, hashing must also use `id` only. Apple’s documentation states that the components used for hashing must match the components compared by `==`. ([Apple Developer](https://developer.apple.com/documentation/swift/hashable/hash%28into%3A%29?utm_source=chatgpt.com "hash(into:) | Apple Developer Documentation"))

### Hash values are not stable identifiers

Do not persist `hashValue`. Do not send it to a backend. Do not use it as a database key. Do not expect it to be stable across app launches.

Apple documents that hash values are not guaranteed to be equal across executions and should not be saved for future execution. ([Apple Developer](https://developer.apple.com/documentation/swift/hasher/finalize%28%29?utm_source=chatgpt.com "finalize() | Apple Developer Documentation"))

Bad:

```swift
let cacheKey = user.hashValue
saveToDisk(cacheKey)
```

Better:

```swift
let cacheKey = user.id
saveToDisk(cacheKey)
```

### `Set` uniqueness depends on `Hashable`

A `Set` is only as correct as the `Hashable` conformance of its element type. The Swift book describes set elements as requiring `Hashable`, with equal values having equal hash values. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/collectiontypes/?utm_source=chatgpt.com "Collection Types - Documentation"))

```swift
struct Tag: Hashable {
    let rawValue: String
}

let tags: Set<Tag> = [
    Tag(rawValue: "swift"),
    Tag(rawValue: "ios"),
    Tag(rawValue: "swift")
]

print(tags.count) // 2
```

---

## 3. Common traps and misconceptions

### Trap 1: Including mutable fields in `hash(into:)`

Bad:

```swift
final class User: Hashable {
    let id: Int
    var name: String

    static func == (lhs: User, rhs: User) -> Bool {
        lhs.id == rhs.id && lhs.name == rhs.name
    }

    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
        hasher.combine(name)
    }

    init(id: Int, name: String) {
        self.id = id
        self.name = name
    }
}

let user = User(id: 1, name: "A")
var users: Set<User> = [user]

user.name = "B" // Mutates hash/equality identity while inside the Set.
```

Better:

```swift
final class User: Hashable {
    let id: Int
    var name: String

    static func == (lhs: User, rhs: User) -> Bool {
        lhs.id == rhs.id
    }

    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }

    init(id: Int, name: String) {
        self.id = id
        self.name = name
    }
}
```

Now `name` can change without changing set identity.

### Trap 2: Thinking hash equality means value equality

This is wrong:

```swift
if a.hashValue == b.hashValue {
    print("same value")
}
```

Hash collisions are allowed. Equal values must hash equally, but equally hashed values do not have to be equal.

Better:

```swift
if a == b {
    print("same value")
}
```

### Trap 3: Using `hashValue` as a persistent ID

Bad:

```swift
struct Product: Hashable {
    let sku: String
    let region: String
}

let persistentID = product.hashValue
```

Better:

```swift
struct Product: Hashable {
    let sku: String
    let region: String

    var persistentID: String {
        "\(region):\(sku)"
    }
}
```

`hashValue` exists for hash-table placement, not durable identity.

### Trap 4: Making equality too broad for an entity

Bad:

```swift
struct User: Hashable {
    let id: Int
    var name: String
    var avatarURL: URL?
    var lastSeenAt: Date?
}
```

Synthesized `Hashable` includes all stored properties. That may be technically valid but semantically wrong. A user’s avatar or last-seen timestamp changing should probably not make them a different user.

Better:

```swift
struct User: Hashable {
    let id: Int
    var name: String
    var avatarURL: URL?
    var lastSeenAt: Date?

    static func == (lhs: User, rhs: User) -> Bool {
        lhs.id == rhs.id
    }

    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }
}
```

---

## 4. Direct answers to rubric questions

### Q1. What invariant must always hold between `==` and `hash(into:)`?

If two values are equal according to `==`, they must feed the same essential components into `Hasher`, in the same order.

Correct:

```swift
struct User: Hashable {
    let id: Int
    var name: String

    static func == (lhs: User, rhs: User) -> Bool {
        lhs.id == rhs.id
    }

    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }
}
```

Incorrect:

```swift
struct User: Hashable {
    let id: Int
    var name: String

    static func == (lhs: User, rhs: User) -> Bool {
        lhs.id == rhs.id
    }

    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
        hasher.combine(name)
    }
}
```

The incorrect version allows two users to be equal by `id` but hash differently because their names differ.

Interview version:

> `Hashable` refines `Equatable`. The required invariant is: if `a == b`, then `a` and `b` must hash using the same essential components. The reverse is not required because hash collisions are valid. In practice, I make `hash(into:)` use exactly the same domain identity as `==`, and I avoid including mutable or incidental fields unless the type is a true immutable value object.

### Q2. Why is storing mutable fields in a `Hashable` key dangerous?

Because `Set` and `Dictionary` place elements using the hash computed at insertion time. If you mutate a field that participates in equality or hashing after insertion, the collection’s internal bucket placement no longer matches the value’s current hash/equality identity.

For value types, this is less likely because the set owns its own stored copy:

```swift
struct User: Hashable {
    var id: Int
    var name: String
}

var user = User(id: 1, name: "A")
var users: Set<User> = [user]

user.name = "B" // Mutates local variable, not the copy inside the Set.
```

But with reference types:

```swift
final class User: Hashable {
    let id: Int
    var name: String

    static func == (lhs: User, rhs: User) -> Bool {
        lhs.id == rhs.id && lhs.name == rhs.name
    }

    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
        hasher.combine(name)
    }

    init(id: Int, name: String) {
        self.id = id
        self.name = name
    }
}

let user = User(id: 1, name: "A")
var users: Set<User> = [user]

user.name = "B" // Mutates the object already stored inside the Set.
```

The set is now structurally inconsistent.

Interview version:

> Mutable hashed state is dangerous because hash-based collections assume the element’s hash/equality identity remains stable while stored. If that identity changes, lookups, removals, duplicate detection, and uniqueness can become wrong. With classes, this is especially subtle because the collection stores a reference, so external mutation changes the object inside the collection without giving the collection a chance to rehash it.

---

## 5. Code probe

Given:

```swift
final class User: Hashable {
    let id: Int
    var name: String

    init(id: Int, name: String) {
        self.id = id
        self.name = name
    }

    static func == (lhs: User, rhs: User) -> Bool {
        lhs.id == rhs.id && lhs.name == rhs.name
    }

    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
        hasher.combine(name)
    }
}

let a = User(id: 1, name: "A")
let b = User(id: 1, name: "B")

var users: Set<User> = [a, b]

a.name = "B"

let values = users.map { "\($0.id)-\($0.name)" }.sorted()

print(users.count)
print(values)
print(Set(values).count)
```

### What happens?

Verified with Swift 6.2.1:

```text
2
["1-B", "1-B"]
1
```

### Why?

Initial state:

```text
a = User(id: 1, name: "A")
b = User(id: 1, name: "B")

a == b // false
```

Because equality uses both `id` and `name`, the set accepts both:

```text
users = { a, b }
users.count = 2
```

Then:

```swift
a.name = "B"
```

Now:

```text
a = User(id: 1, name: "B")
b = User(id: 1, name: "B")

a == b // true
```

But the set already contains both references. It does not automatically remove one. It does not know `a` changed its hash/equality identity.

So this happens:

```text
Set storage:
- element reference a, now displaying "1-B"
- element reference b, displaying "1-B"

users.count = 2
values = ["1-B", "1-B"]
Set(values).count = 1
```

The last line is `1` because the strings are ordinary immutable values, and both strings are equal.

### Why this is a correctness bug even though the code compiles

Swift’s type system can verify that `User` has the required methods for `Hashable`. It cannot prove that `name` will not be mutated while the object is inside a `Set`.

The compiler checks conformance shape:

```text
Does User provide ==?
Does User provide hash(into:)?
```

It does not fully verify semantic stability:

```text
Will fields used by == and hash(into:) remain stable while stored in a Set?
```

That second part is your responsibility.

### Fix or redesign

If `User` is an entity with stable identity, use `id` only:

```swift
final class User: Hashable {
    let id: Int
    var name: String

    init(id: Int, name: String) {
        self.id = id
        self.name = name
    }

    static func == (lhs: User, rhs: User) -> Bool {
        lhs.id == rhs.id
    }

    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }
}
```

Now:

```swift
let a = User(id: 1, name: "A")
let b = User(id: 1, name: "B")

var users: Set<User> = [a, b]

print(users.count)
```

Output:

```text
1
```

Because `a` and `b` represent the same user identity.

### Why this fix is correct

The stable identity of a user is `id`. The display name is mutable presentation/domain data, not collection identity.

Now the invariant is stable:

```text
a == b            iff a.id == b.id
hash(a) == hash(b) if a.id == b.id
```

Changing `name` no longer changes the object’s identity inside hash-based collections.

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|Hash/equality by stable `id` only|Entity models: users, orders, messages, database rows|Two objects with same `id` but different fields collapse into one set element|
|Make all hashed fields immutable|True value objects where all fields define identity|Updates require creating a new value|
|Remove, mutate, reinsert|Rare cases where mutable fields really define identity|Easy to forget; fragile API shape|
|Use a stable key wrapper|You need a set/dictionary keyed by identity while storing mutable objects separately|More types and indirection|
|Avoid storing mutable reference types directly in `Set`|Large codebases where mutation ownership is unclear|Requires clearer architecture|

Example key-wrapper design:

```swift
struct UserID: Hashable {
    let rawValue: Int
}

final class User {
    let id: UserID
    var name: String

    init(id: UserID, name: String) {
        self.id = id
        self.name = name
    }
}

var usersByID: [UserID: User] = [:]

let user = User(id: UserID(rawValue: 1), name: "A")
usersByID[user.id] = user

user.name = "B" // Safe: dictionary key did not change.
```

---

## 6. Exercise

### Problem

Debug a bug where a value disappears from a `Set` after a property mutation.

### Bad / naive version

```swift
final class Item: Hashable {
    let id: Int
    var title: String

    init(id: Int, title: String) {
        self.id = id
        self.title = title
    }

    static func == (lhs: Item, rhs: Item) -> Bool {
        lhs.id == rhs.id && lhs.title == rhs.title
    }

    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
        hasher.combine(title)
    }
}

let target = Item(id: 100, title: "Draft")

var items = Set((0..<1_000).map {
    Item(id: $0, title: "Item \($0)")
})

items.insert(target)

print(items.contains(target)) // true

target.title = "Published"

print(items.contains(target)) // Can be false
print(items.remove(target) as Any) // Can be nil
print(items.contains { $0 === target }) // true
```

### What is wrong?

```text
The Set still contains the object reference.
But the object's hash/equality identity changed after insertion.
Lookup now searches using the new hash, while the object may still live in a bucket chosen from the old hash.
```

This produces the “disappearing element” bug:

```text
The element is physically present.
But normal Set lookup/removal may not find it.
```

### Improved version

```swift
final class Item: Hashable {
    let id: Int
    var title: String

    init(id: Int, title: String) {
        self.id = id
        self.title = title
    }

    static func == (lhs: Item, rhs: Item) -> Bool {
        lhs.id == rhs.id
    }

    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }
}

let target = Item(id: 100, title: "Draft")

var items = Set((0..<1_000).map {
    Item(id: $0, title: "Item \($0)")
})

items.insert(target)

print(items.contains(target)) // true

target.title = "Published"

print(items.contains(target)) // true
print(items.remove(target) as Any) // Optional(...)
```

### Why this is better

The set identity is stable. `title` can change without invalidating the hash table. The collection answers identity questions by `id`, not by mutable display state.

For app code, this is usually what you want for entities:

```text
User identity: id
Order identity: id
Message identity: id
Product identity: sku or backend id
View model identity: stable model id, not display fields
```

For true value objects, prefer immutability:

```swift
struct Coordinate: Hashable {
    let latitude: Double
    let longitude: Double
}
```

If the fields define identity, they should usually be immutable while used in a set.

---

## 7. Production guidance

Use this in production when:

```text
You need stable identity for Set elements or Dictionary keys.
You model entities from a backend or database.
You implement diffing, selection state, caches, de-duplication, or lookup tables.
You need predictable collection behavior across mutations.
```

Be careful when:

```text
Hashable is synthesized for models with mutable fields.
A class conforms to Hashable.
An object is stored in Set or used as Dictionary key.
A field used by == or hash(into:) can change.
Equality means different things in different parts of the app.
```

Avoid when:

```text
You want a persistent ID: do not use hashValue.
You want cryptographic hashing: Hashable is not that.
You want object identity: use ObjectIdentifier or === deliberately.
You want database identity: use a real stable key.
```

Debugging checklist:

```text
Is this type a value object or entity?
Which fields define equality?
Are those exact fields used in hash(into:)?
Can any equality/hash field mutate while stored in Set or Dictionary?
Is Hashable synthesized accidentally including fields like name, timestamp, imageURL, loading state, or cache?
Is a class used as a Set element or Dictionary key?
Would a stable ID wrapper be clearer?
Are we confusing hashValue with persistent identity?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> `Hashable` lets you put things in sets and dictionaries. Equal things should have the same hash.

### Senior answer

> `Hashable` is a semantic contract with `Equatable`. The fields used by `hash(into:)` must match the fields used by `==`, and those fields must remain stable while the value is used as a set element or dictionary key. Mutable reference types are risky because the collection stores the reference and cannot observe mutations to equality/hash-relevant fields.

### Staff-level answer

> I separate entity identity from mutable state. For entities, I usually define equality and hashing by a stable ID only, or I avoid making the mutable object itself `Hashable` and use a key wrapper like `UserID`. For value objects, I prefer immutable fields and synthesized conformance when the stored properties exactly match semantic equality. I also treat `Hashable` as an API design decision: once clients rely on equality, changing it can break diffing, caches, selection, persistence migrations, and collection behavior.

Staff-level questions to ask:

```text
Is this type an entity or a value object?
Does synthesized Hashable match the domain meaning of equality?
Can any hash-relevant property mutate after insertion into a collection?
Would a stable ID type make the API safer?
Will changing equality semantics break diffing, caching, persistence, or UI selection?
```

---

## 9. Interview-ready summary

`Equatable` defines semantic sameness; `Hashable` makes that sameness usable by hash-based collections. The critical invariant is that equal values must hash from the same essential components. Hash collisions are fine, but mismatched equality and hashing are not. The production trap is mutable hashed state, especially on classes stored in `Set` or used as `Dictionary` keys. If a field used by `==` or `hash(into:)` changes after insertion, the collection’s internal hash-table invariants are broken. For entities, prefer stable ID-based equality and hashing. For true value objects, make identity-defining fields immutable.

---

## 10. Flashcards

Q: What invariant must hold between `==` and `hash(into:)`?  
A: If `a == b`, then `a` and `b` must feed the same essential components into `Hasher`, in the same order.

Q: Does equal hash imply `==`?  
A: No. Hash collisions are allowed. Equal values must hash equally, but equally hashed values do not have to be equal.

Q: Why is mutable hashed state dangerous?  
A: A `Set` or `Dictionary` places an element/key using its hash at insertion time. Mutating hash/equality-relevant fields afterward can make lookup, removal, or uniqueness incorrect.

Q: Why are classes especially risky as `Hashable` set elements?  
A: A set stores references to class instances. External mutation changes the object inside the set without rehashing it.

Q: Should `hashValue` be persisted?  
A: No. Hash values are not stable across executions and are not persistent identifiers.

Q: For an entity like `User`, what should usually define equality?  
A: A stable identity such as `id`, not mutable display fields like `name` or `avatarURL`.

Q: When is synthesized `Hashable` good?  
A: When all stored properties exactly match the type’s semantic equality and are stable enough for intended collection use.

Q: How do you fix a set element that must change a hash-relevant field?  
A: Remove it before mutation and reinsert it after mutation, or redesign so mutable fields do not participate in hashing.

---

## 11. Related sections

- [[B7 — Synthesized conformances and semantic correctness]]
- [[A1 — Value semantics, reference semantics, identity, and copy-on-write]]
- [[A15 — Type choices - struct, class, enum, protocol, and actor]]
- [[C2 — Copy-on-write behavior of standard-library types]]
- [[C5 — Collection protocols, complexity, and index invalidation]]
- [[B11 — Library evolution and resilience-aware API design]]

---

## 12. Sources

- Swift Senior/Staff Rubric and Prioritized Study Checklist, B8 — Equatable, Hashable, and collection correctness.
- Apple Developer Documentation — `Hashable`: equal instances must feed the same values to `Hasher` in the same order. ([Apple Developer](https://developer.apple.com/documentation/Swift/Hashable?utm_source=chatgpt.com "Hashable | Apple Developer Documentation"))
- Apple Developer Documentation — `hash(into:)`: hash the same components compared by `==`. ([Apple Developer](https://developer.apple.com/documentation/swift/hashable/hash%28into%3A%29?utm_source=chatgpt.com "hash(into:) | Apple Developer Documentation"))
- Apple Developer Documentation — `Hasher.finalize()`: hash values are not guaranteed across executions and should not be saved. ([Apple Developer](https://developer.apple.com/documentation/swift/hasher/finalize%28%29?utm_source=chatgpt.com "finalize() | Apple Developer Documentation"))
- The Swift Programming Language — Collection Types: set elements require `Hashable`, and equal values must have equal hash values. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/collectiontypes/?utm_source=chatgpt.com "Collection Types - Documentation"))