# Dart for Serialization (Proposal)

Design and open _discussion_ around better Dart serialization experiences.

_This is **not** an official Google or Dart design or process is primarily to
experiment and collect feedback from both internal and external stakeholders
and users. There is a chance this might result in no changes, or changes could
occur at some future point in time._

**Please read [CONTRIBUTING.md](CONTRIBUTING.md) before opening a PR**

**Table of contents**

* [Background](#background)
* [Available Options](#available-options)
  * [Use untyped data structures](#use-untyped-data-structures)
  * [Use runtime reflection](#use-runtime-reflection)
  * [Use code generation](#use-code-generation)
  * [Hand written classes](#hand-written-classes)
* [Problems](#problems)
  * [Ergonomics](#ergonomics)
  * [Extensibility](#extensibility)
  * [Performance](#performance)
* [Stakeholders](#stakeholders)
  * [Dart language team](#dart-language-team)
  * [Dart platform team](#dart-platform-team)
  * [Dart web users](#dart-web-users)
  * [Flutter users](#flutter-users)
* [References](#references)
  * [TypeScript](#typescript)
  * [Swift](#swift)
  * [Kotlin](#kotlin)
  * [C#](#c)
  * [Rust](#rust)
  * [Go](#go)
* [Solutions](#solutions)
  * [Language](#language)
  * [Platform](#platform)
  * [Packages](#packages)

## Background

Dart is toted as a modern object-oriented client-side programming language
(with some but more limited usage in standalone tools and servers), but lacks a
well defined story for serialization - something that is paramount to adoption
and healthy developer ergonomics.

With the introduction of [Flutter][], Dart users have the ability to produce
high-fidelity iOS and Android applications, but have also run into the issue of
having a typed object model that serializes to/from data formats like JSON.

[Flutter]: https://flutter.io

> A prior review of options in Dart that is a bit of out date is available on the
> Dart website as an article: ["Serialization in Dart"][dart_article]. It's a good
> place to start if you don't know much about the subject.

[dart_article]: https://www.dartlang.org/articles/libraries/serialization

## Available Options

Not surprisingly, users _are_ effectively using Dart with JSON and other
formats like protocol buffers, today, both internally and externally, but often
with some sort of downside.

### Use untyped data structures

The simplest option - just don't use types. This often produces the lowest
overhead in terms of both runtime and code-size, and can even scale OK for
small projects or in cases where the serialization format is fluid.

```dart
import 'dart:async';
import 'dart:convert';

import 'package:http/http.dart' as http;

Future<Map<String, dynamic>>> fetchAccount(int id) async {
  var response = await http.get('account/$id');
  return JSON.decode(response.body) as Map<String, dynamic>;
}
```

```dart
main() async {
  var account = await fetchAccount(101);
  print('Account #101 is for: ${account['name']'});
}
```

####  PROs
* Batteries included: you rarely have to go beyond the core libraries.
* Works equally well in all platforms (browser/server/mobile).
* Produces the lowest overhead in terms of both code-size and runtime.

#### CONs
* Toolability: Good luck renaming `account['name']` to `account['first_name']`.
* Can't statically validate against a schema or format.
* Doesn't scale well to large teams. 
* Exposes a mutable and iterable data structure for everything.

#### Who is using this approach
* Small teams or single developers/prototypers.
* Applications with unstructured data (so `Map` or `List` is actually OK).

### Use runtime reflection

Dart's runtime reflection library ([`dart:mirrors`][mirrors]) can read from and
write to structured classes at runtime, and use metadata like annotations for
developers to be able to add extra information.

[mirrors]: https://www.dartlang.org/articles/libraries/reflection-with-mirrors

```dart
import 'dart:convert';
import 'dart:mirrors';

class Account {
  final int id;

  Account({this.id});
}

class Serializer<T> {
  const Serializer<T>();

  T decode(String json) {
    var type = reflectType(T);
    var constructor = type.constructors.first;
    var parameters = constructor.namedParameters;
    return type.newInstance(
      namedParameters: JSON.decode(json),
    );
  }
}

main() {
  var account = const Serializer<Account>().decode(r'''
    {
      "id": 101
    }
  ''');
  print('Account #101 is for: ${account.id'});
}
```

#### PROs
* Batteries mostly included: trivial to write a simple serialization library.
* Potential to remove mirrors usage using source code transformation.

#### CONs
* Arbitrary requirements on classes (public constructor, etc).
* Serious platform issues: disabled for Flutter, unsuable code size on the web.
* Runtime performance suffers compared to statically typed code.
* Difficult to debug: reflection-based systems harder to reason about.
* Source code transformation is mostly terrible for good build systems.

#### Who is using this approach
* [`package:serialization`](https://pub.dartlang.org/packages/serialization)
* [`package:dartson`](https://pub.dartlang.org/packages/dartson)

### Hand written classes

Of course, hand-written code and classes can do precisely what you want. For
small projects or systems that don't change often (or for precisely optmizing
for your business requirements) this might make the most sense:

```dart
import 'dart:convert';

class Account {
  final int id;

  Account({this.id});
}

class AccountSerializer {
  const AccountSerializer();

  Account decode(String json) {
    var map = JSON.decode(json) as Map<String, dynamic>;
    return new Account(id: map['id']);
  }
}

main() {
  var account = const AccountSerializer().decode(r'''
    {
      "id": 101
    }
  ''');
  print('Account #101 is for: ${account.id'});
}
```

#### PROs
* You get exactly the behavior and code you want.

#### CONs
* Any medium-sized+ data model is going to be time consuming/error prone.
* As the data model changes must change both class and serializer.
* Hard to ever support other data formats with writing even more code.
* Dart feels like it has platform/language issues to those who read the code.

#### Who is using this approach
* Too many to count :)

### Use code generation

A time-tested option, simply generate Dart code either ahead-of-time or during
the development process from another data source, such as a schema,
configuration file, or even Dart source code (static source analysis).

Most internal users at Google use this strategy (in some form or another,
though the most common is protocol buffers) - but this also relies on fact we
have [bazel](https://bazel.build) as a standard build system.

```dart
import 'package:my_json_generator/my_json_generator.dart';

part 'main.g.dart';

class Account {
  final int id;

  Account({this.id});
}

@generate
abstract class AccountSerializer {
  const factory AccountSerializer() = AccountSerializer$Generated;

  Account decode(String json);
}

main() {
  var account = const AccountSerializer).decode(r'''
    {
      "id": 101
    }
  ''');
  print('Account #101 is for: ${account.id'});
}
```

#### PROs
* Nicer ergonomics compared to hand writing (once build system in place).
* Possible to get very close to (depending on requirements) hand-written code.
* Getting more popular in web community with introduction of CLIs.

#### CONs
* Difficulty of designing the "perfect system" (definition varies).
* Static analysis errors: until you generate `main.g.dart`, at least.
* Dart lacks a complete standard build system that works equally well for all.
* For frameworks like Flutter that are "batteries included", this falls short.

#### Who is using this approach
* [Protocal buffers](https://github.com/dart-lang/dart-protoc-plugin)
* [`package:serializable`](https://pub.dartlang.org/packages/serializable)
* [`package:build_value`](https://pub.dartlang.org/packages/built_value)
* [`package:owl`](https://pub.dartlang.org/packages/owl)
* [`package:streamy`](https://pub.dartlang.org/packages/streamy)

## Problems

### Ergonomics

### Extensibility

### Performance

## Stakeholders

Who (entities or individuals) are effected or have a business need in this
proposal. Note that being a _user_ of serialization is likely not enough to be
considered a stakeholder - though the goal of this document is to solicit
_feedback_ from indidiual users.

**NOTE**: The following list is entirely assumptive at this moment:

### Dart language team

#### Requirements

TBD

### Dart platform team

#### Requirements

TBD

### Dart web users

#### Requirements

TBD

### Flutter users

#### Requirements

TBD

## References

> **NOTE**: Languages with dynamic typing without any form of static analysis
> (i.e. JavaScript, Ruby, Python) are excluded - in-that often the platform's
> built-in serialization is enough to avoid needing anything else.
>
> It's also not that interesting to compare against - and optimizers like
> Google's [closure][] often have different requirements than the language
> itself.

[closure]: https://developers.google.com/closure/compiler/

### TypeScript

TypeScript has a structural type system that is easy to overlay ontop of
(unstructed) formats like JSON. As an example we can pretend that a received
JSON blob representing a `User` has static types:

```ts
interface User {
  name:    string;
  age:     number;
  created: Date;
}

fetchById(id: int): Promise<User> {
  return http.get('/users/${id}').map((response) => JSON.parse(response));
}

function run() {
  fetchById(101).then((user) => {
    console.log('Hello! ${user.name} is ${user.age} year(s) old.');
  });
}
```

If _decorators_ are introduced ([see proposal][ts_decorators]) then a
lightweight macro-like syntax will exist as well to generate boilerplate. As an
example:

[ts_decorators]: https://github.com/wycats/javascript-decorators

```ts
@serializable()
class User {
    constructor(name: string) {
      this._name = name;
    }

    private _name: string;

    @serialize()
    get name() {
      return this._name;
    }
}

function run() {
  const p = new Person('André'); 
  console.log(JSON.stringify(p));
}
```

### Swift

Using the [`Gloss`](http://harlankellaway.com/Gloss/) library you get some
helper functions and syntax, but nothing too magical. 

```swift
import Gloss

struct RepoOwner: Decodable {

    let ownerId: Int?
    let username: String?

    // MARK: - Deserialization

    init?(json: JSON) {
        self.ownerId = "id" <~~ json
        self.username = "login" <~~ json
    }
}
```

Or for translating to JSON:

```swift
import Gloss

struct RepoOwner: Glossy {

    let ownerId: Int?
    let username: String?

    // MARK: - Deserialization
    // ...

    // MARK: - Serialization

    func toJSON() -> JSON? {
        return jsonify([
            "id" ~~> self.ownerId,
            "login" ~~> self.username
        ])
    }
}
```

### Kotlin

An example of the [`Kotson`][https://github.com/SalomonBrys/Kotson] library:

```kt
import com.github.salomonbrys.kotson.*

val gson = GsonBuilder().registerTypeAdapter<Person>(personSerializer).create()
```

```kt
import com.github.salomonbrys.kotson.*

val gson = Gson()

// java: List<User> list = gson.fromJson(src, new TypeToken<List<User>>(){}.getType());
val list1 = gson.fromJson<List<User>>(jsonString)
val list2 = gson.fromJson<List<User>>(jsonElement)
val list3 = gson.fromJson<List<User>>(jsonReader)
val list4 = gson.fromJson<List<User>>(reader)
```

### C#

An example of the [`ServiceStack`][ServiceStack] library:

[ServiceStack]: https://github.com/ServiceStack/ServiceStack.Text

(You can try this example [live in your browser][live_example])

[live_example]: http://gistlyn.com/?gist=52c37e37b51a0ec92810477be34695ae&collection=2cc6b5db6afd3ccb0d0149e55fdb3a6a

```cs
using System.Linq;
using ServiceStack;
using ServiceStack.Text;

public class GithubRepository
{
    public string Name { get; set; }
    public string Description { get; set; }
    public string Url { get; set; }
    public string Homepage { get; set; }
    public string Language { get; set; }
    public int Watchers { get; set; }
    public int Forks { get; set; }
    
    public override string ToString() => Name;
}

var orgName = "ServiceStack";

var orgRepos = $"https://api.github.com/orgs/{orgName}/repos"
    .GetJsonFromUrl(httpReq => httpReq.UserAgent = "Gistlyn")
    .FromJson<GithubRepository[]>()
    .OrderByDescending(x => x.Watchers)
    .Take(5)
    .ToList();

"Top 5 {0} Github Repositories:".Print(orgName);
orgRepos.PrintDump();

// Save a copy of this *public* Gist by clicking the "Save As" below 
```

### Rust

TODO.

### Go

Go comes with the [`encoding/json`](https://golang.org/pkg/encoding/json/) package in the standard library which allows serialization and deserialization of data types.

Here's an example: 

(you can try this example [live in your browser](https://play.golang.org/p/BErOzJz5dq))

```go
package main

import (
    "encoding/json"
    "fmt"
)

const personJSON = `{
  "Name" : "John Doe",
  "Age" : 50
}`

func main() {
    // deserialize a JSON string to a 'person' type
    deserializedPerson := deserializePerson(personJSON)
    fmt.Printf("Deserialized: %+v\n\n", deserializedPerson)
    
    // serialize a 'person' type to a JSON string
    serializedPerson := serializePerson(deserializedPerson)
    fmt.Printf("Serialized: %s", serializedPerson)

}

type person struct {
    Name string
    Age  int
}

func deserializePerson(s string) person {
    var p person
    json.Unmarshal([]byte(s), &p)
    return p
}

func serializePerson(p person) string {
    b, _ := json.Marshal(p)
    return string(b)
}
```


## Solutions

Below are partial, theoritical, and non-exhaustive potential solutions to help
make Dart's serialization and JSON story better. Nothing has been agreed on -
and we're open to other ideas. Likely the "solution" will involve _many_ things
- not just one.

### Language

#### Allow invoking a constructor using JSON/Map

Users write code like this:

```dart
class User {
  final int id;

  User({this.id});
}
```

And can invoke `User.new` (and named parameters using a JSON/Map):

```dart
main() {
  new User(~{'id': 5});
}
```

#### Add anonymous classes

While this doesn't strictly fix the JSON/serialization issues, it does make an
abstract class/interface be a more implicitly useful data model. For example:

```dart
main() {
  var user = new User {id: 5};
}
```

#### Add a new `struct` datatype with a different type system

```dart
struct User {
  int id;
}
```

And perhaps it could participate in a structural type system where something
like a `Map` or `JsObject` (for web users) could be "cast" (represented as)
this struct:

```dart
main() {
  var json = {id: 5} as User;
  print(json.id);
}
```

#### Add a new `Json` union type

```dart
typedef Json = String | num | List<Json> | Map<Json, Json> | Null;
```

#### Add macros

TODO

#### Invest in better `dart:mirrors`

TODO

### Platform

I.e. something that can be better optimized/tree-shaken across platforms.

#### Invest in an universal build system for Dart packages

A seamless code generation story, perhaps with better analyzer integration,
incremental builds, and IDE-awareness would allow the least amount of changes
to the language. We would likely still need a canonical serialization library.

#### Allow `dart2js` to optimize away "wrapper" code

TODO

### Packages

#### Provde a canonical serialization library on top of code generation

TODO
