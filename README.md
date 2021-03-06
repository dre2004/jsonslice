[![Build Status](https://travis-ci.org/bhmj/jsonslice.svg?branch=master)](https://travis-ci.org/bhmj/jsonslice)
[![Go Report Card](https://goreportcard.com/badge/github.com/bhmj/jsonslice)](https://goreportcard.com/report/github.com/bhmj/jsonslice)
[![GoDoc](https://godoc.org/github.com/bhmj/jsonslice?status.svg)](https://godoc.org/github.com/bhmj/jsonslice)


# JSON Slice

## What is it?

JSON Slice is a Go package which allows to execute fast jsonpath queries without unmarshalling the whole data.  

Sometimes you need to get a single value from incoming json using jsonpath, for example to route data accordingly or so. To do that you must unmarshall the whole data into interface{} struct and then apply some jsonpath library to it, only to get just a tiny little value. What a waste of resourses! Well, now there's `jsonslice`.

Simply call `jsonslice.Get` on your raw json data to slice out just the part you need. The `[]byte` received can then be unmarshalled into a struct or used as it is.

## Getting started

#### 1. install

```
$ go get github.com/bhmj/jsonslice
```

#### 2. use it

```golang
import "github.com/bhmj/jsonslice"
import "fmt"

func main() {
    var data = []byte(`
    { "sku": [ 
        { "id": 1, "name": "Bicycle", "price": 160, "extras": [ "flashlight", "pump" ] },
        { "id": 2, "name": "Scooter", "price": 280, "extras": [ "helmet", "gloves", "spare wheel" ] }
      ]
    } `)

    a, _ := jsonslice.Get(data, "$.sku[0].price")
    b, _ := jsonslice.Get(data, "$.sku[1].extras.count()")
    c, _ := jsonslice.Get(data, "$.sku[?(@.price > 200)].name")
    d, _ := jsonslice.Get(data, "$.sku[?(@.extras.count() < 3)].name")

    fmt.Println(string(a)) // 160
    fmt.Println(string(b)) // 3
    fmt.Println(string(c)) // ["Scooter"]
    fmt.Println(string(d)) // ["Bicycle"]
}
```
[Run in Go Playground](https://play.golang.org/p/fYv-Y12akvs)

## Package functions
  
`jsonslice.Get(data []byte, jsonpath string) ([]byte, error)`  
  - get a slice from raw json data specified by jsonpath

`jsonslice.GetArrayElements(data []byte, jsonpath string, alloc int) ([][]byte, error)`
  - get a slice of array elements from raw json data specified by jsonpath

## Benchmarks (Core i5-7500)

```diff
$ go test -bench=. -benchmem -benchtime=4s
goos: linux
goarch: amd64
pkg: github.com/bhmj/jsonslice
++ here's a couple of operations usually needed to get an object by jsonpath (for reference):
Benchmark_Unmarshal-4                     500000             15487 ns/op            4496 B/op        130 allocs/op
Benchmark_Oliveagle_Jsonpath-4           3000000              1981 ns/op             608 B/op         48 allocs/op
++ and here's a jsonslice.Get:
Benchmark_Jsonslice_Get-4                2000000              2840 ns/op              32 B/op          1 allocs/op
++ Get() involves parsing a jsonpath, here it is:
Benchmark_JsonSlice_ParsePath-4         10000000               579 ns/op               0 B/op          0 allocs/op
++ in case you aggregate some non-contiguous elements, it may take a bit longer (extra mallocs involved):
Benchmark_Jsonslice_Get_Aggregated-4     1000000              5853 ns/op            2178 B/op         10 allocs/op
++ unmarshalling a large json:
Benchmark_Unmarshal_10Mb-4                   100          47716510 ns/op             376 B/op          5 allocs/op
++ jsonslicing the same json, target element is near the start:
Benchmark_Jsonslice_Get_10Mb_First-4     5000000              1312 ns/op              32 B/op          1 allocs/op
++ jsonslicing the same json, target element is near the end: still beats Unmarshal
Benchmark_Jsonslice_Get_10Mb_Last-4          200          29554432 ns/op              37 B/op          1 allocs/op
PASS
ok      github.com/bhmj/jsonslice       71.377s

```

## Specs

See [Stefan Gössner's article](http://goessner.net/articles/JsonPath/index.html#e2) for original specs and examples.  

## Limitations and deviations

1. Single-word keys (`/\w+/`) are supported in dot notation mode; use bracket notation for multi-word keys.

2. A single index reference returns an element, not an array:  
```
./jsonslice '$.store.book[0]' sample0.json
```
returns  
```
{
  "category": "reference",
  "author": "Nigel Rees",
  "title": "Sayings of the Century",
  "price": 8.95
}
```
while this query
```
./jsonslice '$.store.book[0:1]' sample0.json
```
returns an array 
```
[{
  "category": "reference",
  "author": "Nigel Rees",
  "title": "Sayings of the Century",
  "price": 8.95
}]
```

Also, indexing on root node is supported (assuming json is an array and not an object):  
```
./jsonslice '$[0].author' sample1.json
```

## Expressions

### Common expressions

#### Operators 
```
  $                   -- root node (can be either object or array)
  .node               -- dot-notated child
  ['node']            -- bracket-notated child
  ['foo','bar']       -- bracket-notated children
  [123]               -- array index
  [12:34]             -- array range
```
#### Functions
```
  $.obj.length()      -- number of elements in an array or string length, depending on the obj type
  $.obj.count()       -- same as above
  $.obj.size()        -- object size in bytes (as is)
```
#### Objects
```
  $.obj
  $.obj.val
  $.*                 -- wildcard (matches any value of any type)
  $.*.val             -- wildcard object (matches any object)
  $.*[:].val          -- wildcard array (matches any array)
```
####  Indexed arrays
```
  $.obj[3]
  $.obj[3].val
  $.obj[-2]  -- second from the end
```
#### Ranged arrays
```
  $.obj[:]   -- == $.obj (all elements of the array)
  $.obj[0:]  -- the same as above: items from index 0 (inclusive) till the end
  $.obj[<anything>:0] -- doesn't make sense (from some element to the index 0 exclusive -- which is always empty)
  $.obj[2:]  -- items from index 2 (inclusive) till the end
  $.obj[:5]  -- items from the beginning to index 5 (exclusive)
  $.obj[-2:] -- items from the second element from the end (inclusive) till the end
  $.obj[:-2] -- items from the beginning to the second element from the end (exclusive, i.e. without two last elements)
  $.obj[:-1] -- items from the beginning to the end but without one final element
  $.obj[2:5] -- items from index 2 (inclusive) to index 5 (exclusive)
```

### Aggregating expressions

#### Sub-querying
```
  $.obj[any:any].something  -- composite sub-query
  $.obj[3,5,7]              -- multiple array indexes
```
#### Filters
```
  [?(<expression>)]  -- filter expression. Applicable to arrays only
  @                  -- the root of the current element of the array. Used only within a filter.
  @.val              -- a field of the current element of the array.
```

#### Filter operators

  Operator | Description
  --- | ---
  `==`  | Equal to<br>Use single or double quotes for string expressions.<br>`[?(@.color=='red')]` or `[?(@.color=="red")]`
  `!=`  | Not equal to<br>`[?(@.author != "Herman Melville")]`
  `>`   | Greater than<br>`[?(@.price > 10)]`
  `>=`  | Grater than or equal to
  `<`   | Less than
  `<=`  | Less than or equal to
  `=~`  | Match a regexp<br>`[?(@.name =~ /sword.*/i]`
  `&&`  | Logical AND<br>`[?(@.price < 10 && @isbn)]`
  `\|\|`  | Logical OR<br>`[?(@.price > 10 \|\| @.category == 'reference')]`

"Having" filter:  
`$.stores[?(@.work_time[:].time_close=="16:00:00")])].id` -- find IDs of every store having at least one day with a closing time at 16:00

### Updates (TODO)

```
  $.obj[?(@.price > 1000)].expensive = true                    -- add/replace field value
  $.obj[?(@.authors.size() > 2)].title += " (multi authored)"  -- expand field value
  $.obj[?(@.price > $.expensive)].bonus = $.bonuses[0].value   -- add/replace field using another jsonpath 
```

## Examples

  Assuming `sample0.json` and `sample1.json` in the example directory:  

  `cat sample0.json | ./jsonslice '$.store.book[0]'`  
  `cat sample0.json | ./jsonslice '$.store.book[0].title'`  
  `cat sample0.json | ./jsonslice '$.store.book[0:-1]'`  
  `cat sample1.json | ./jsonslice '$[1].author'`  
  `cat sample0.json | ./jsonslice '$.store.book[?(@.price > 10)]'`  
  `cat sample0.json | ./jsonslice '$.store.book[?(@.price > $.expensive)]'`  

  More examples can be found in `jsonslice_test.go`  
  
## Changelog

**0.7.5** (2019-05-21) -- Functions `count()`, `size()`, `length()` work in filters.
> `$.store.bicycle.equipment[?(@.count() = 2)]` -> `[["light saber", "apparel"]]`  

**0.7.4** (2019-03-01) -- Mallocs reduced (see Benchmarks section).

**0.7.3** (2019-02-27) -- `GetArrayElements()` added.

**0.7.2** (2018-12-25) -- bugfix: closing square bracket inside a string value.

**0.7.1** (2018-10-16) -- bracket notation is now supported.
> `$.store.book[:]['price','title']` -> `[[8.95,"Sayings of the Century"],[12.99,"Sword of Honour"],[8.99,"Moby Dick"],[22.99,"The Lord of the Rings"]]`

**0.7.0** (2018-07-23) -- Wildcard key (`*`) added.
> `$.store.book[-1].*` -> `["fiction","J. R. R. Tolkien","The Lord of the Rings","0-395-19395-8",22.99]`  
> `$.store.*[:].price` -> `[8.95,12.99,8.99,22.99]`

**0.6.3** (2018-07-16) -- Boolean/null value error fixed.

**0.6.2** (2018-07-03) -- More tests added, error handling clarified.

**0.6.1** (2018-06-26) -- Nested array indexing is now supported.
> `$.store.bicycle.equipment[1][0]` -> `"peg leg"`

**0.6.0** (2018-06-25) -- Regular expressions added.
> `$.store.book[?(@.title =~ /(dick)|(lord)/i)].title` -> `["Moby Dick","The Lord of the Rings"]`

**0.5.1** (2018-06-15) -- Logical expressions added.
> `$.store.book[?(@.price > $.expensive && @.isbn)].title` -> `["The Lord of the Rings"]`

**0.5.0** (2018-06-14) -- Expressions added.
> `$.store.book[?(@.price > $.expensive)].title` -> `["Sword of Honour","The Lord of the Rings"]`

**0.4.0** (2018-05-16) -- Aggregating sub-queries added.
> `$.store.book[1:3].author` -> `["John","William"]`

**0.3.0** (2018-05-05) -- Beta  

## Roadmap

- [x] length(), count(), size() functions
- [x] filters: simple expressions
- [x] filters: complex expressions (with logical operators)
- [x] nested arrays support
- [x] wildcard operator (`*`)
- [ ] deepscan operator (`..`)
- [x] bracket notation for multiple field queries
- [ ] assignment in query (update json)

## Contributing

1. Fork it!
2. Create your feature branch: `git checkout -b my-new-feature`
3. Commit your changes: `git commit -am 'Add some feature'`
4. Push to the branch: `git push origin my-new-feature`
5. Submit a pull request :)

## Licence

[MIT](http://opensource.org/licenses/MIT)

## Author

Michael Gurov aka BHMJ
