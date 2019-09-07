# How I implemented the JSON specification in Go and what I learned

Why bother?
Deliberate practice. I have read and thought a lot about software engineering but I realise that to really gain insights I need to try out the ideas I have come across. The JSON specification is ideal because compared to a real project that has varying requirements and with pressures. This is very well defined and relatively simple. It is also so I could practise Go. I should be a lot more confident about Go by the end of this. So let’s dive in!

## The Process
# Step 1 - MVP

The first step is to focus on getting the structure right and ignore some details. This meant that I initially didn’t support numbers, literals(null, true, false), or string escaping. It started with this function.
```go
func Load(s string) interface{} {
}
```
This is what the user is going to call to convert a string written in JSON to an object
Next was to determine what value is represented by the json. This can be a string, an array or an object and lucky for us in Json there is a simple rule that indicates which of these it is.
If it’s a string, the first non-whitespace character is ‘“‘
if it’s an array, the first non-whitespace character is ‘[‘
If it’s an object, the first non-whitespace character is ‘{‘
The other thing to realise is that a value can occur at any index of s so Load use a helper function load. Putting all of these together we have
```go
package main

//Load converts from string to object using json grammar
func Load(s string, current int) interface{} {
    object, _ := load(s, 0)
    return object
}
func load(s string, current int) (interface{}, int) {
    switch {
    case s[current] == '"':
        return loadString(s, current)
    case s[current] == '[':
        return loadArray(s, current)
    case s[current] == '{':
        return loadObject(s, current)
    }
}

func loadString(s string, current int) (interface{}, int) {
    return "", 0
}

func loadArray(s string, current int) ([]interface{}, int) {
    return make([]interface{}, 0), 0
}

func loadObject(s string, current int) (map[string]interface{}, int) {
    return make(map[string]interface{}), 0
}
```
Next we can focus on the simplest part, parsing a string - this is just read string, parse till the next ‘“‘ is found and that’s it. To parse an array, we need to read in values, separated by “,”. To parse an object, we need to read a value then a ‘:’, then another value and then a comma and do the first step again or we have reached the end of the object. Putting all of that together, we have
```go
package main

import (
    "fmt"
)

//Load converts from string to object using json grammar
func Load(s string, current int) interface{} {
    value, _ := load(s, 0)
    return value
}
func load(s string, current int) (interface{}, int) {
    switch {
    case s[current] == '"':
        return loadString(s, current)
    case s[current] == '[':
        return loadArray(s, current)
    case s[current] == '{':
        return loadObject(s, current)
    default:
        panic(fmt.Sprintf("Unknown value type at %d", current))
    }
}

func loadString(s string, current int) (interface{}, int) {
    current++ // move past the beginning quote
    start := current // first non-quote character
    for current < len(s) && s[current] != '"' {
        current++
    }
    current++ // move past the end quote
    return s[start: current-1], current
}

func loadArray(s string, current int) ([]interface{}, int) {
    array := make([]interface{}, 0)
    current ++ // move past the '['
    for current < len(s) && s[current] != ']' {
        value, current := load(s, current)
        array = append(array, value)
        if s[current] == ']' {
            break
        }
        current++ // move past the ','
    }
    current ++ // move past the ']'
    return array, current
}

func loadObject(s string, current int) (map[string]interface{}, int) {
    object := make(map[string]interface{}, 0)
    current ++ // move past the '['
    for current < len(s) && s[current] != '}' {
        key, current := load(s, current)
        current++ // move past the ':'
        value, current := load(s, current)
        object[key.(string)] = value
        if s[current] == '}' {
            break
        }
        current++ // move past the ','
    }
    current ++ // move past the '}'
    return object, current
}

```
And this should work for basic json - bear in mind, it does no error checking and it doesn’t support whitespace at all except within a string. This was roughly the state at this commit - https://github.com/opethe1st/GoJson/commit/e0dc214e84a8e40d4607b7cf7b0a115b8222ff11 plus tests - tests are vital!