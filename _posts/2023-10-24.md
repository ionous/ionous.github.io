---
layout: post
title: "Go for godot"
date: 2023-10-24
comments: true
---

### Background

I've been wanting to integrate [Tapestry](https://git.sr.ht/~ionous/tapestry) with a game engine for a while now. Since I often use Unreal when [contracting](https://www.linkedin.com/in/ionous), my original thought had been Unity ( it's always nice to try something new ), but with every going on *there* lately, i settled on [Godot](https://godotengine.org/). 


While Tapestry is written in [Go](https://go.dev/), Godot -- despite the name 😉 -- is not. It's written in C, supports C# via Mono, and has its own [GDScript](https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_basics.html) language.


The main options for crossing that language boundary are: 

1. porting
3. process hoisting ( communicating via sockets )
2. cross-compiling


**Porting** is both fragile and time consuming. **Hoisting** is easy, but would have significant runtime overhead. **Cross-compiling** is goldilocks. For [Alice](https://evermany.itch.io/alice) ( which ran in the browser ) i used [GopherJS](https://github.com/gopherjs/gopherjs). For Godot, the options are either: [CGo](https://go.dev/blog/cgo) and direct linking; or, [WebAssembly](https://developer.mozilla.org/en-US/docs/WebAssembly) running on Mono. 

CGo is the simplest route.

Steps
-----

1. Set up the extension
1. Create a bridge for go and godot
1. Compile the go code using cgo
1. Compile the godot extension using scons
1. Loading and use the extension


#### Setting up the extension

Godot has a [template](https://github.com/godotengine/godot-cpp) for creating an extension.

Basically, all that's needed is to download the template and customize it. Their [instructions](https://docs.godotengine.org/en/stable/tutorials/scripting/gdextension/gdextension_cpp_example.html#doc-gdextension-cpp-example)  are pretty good, so no need to repeat it all here.

#### Creating a bridge

To talk between godot and go, first we need a bridge: code that interfaces with Godot on one side, and Golang on the other. As a first attempt, this bridge sends json back and forth using the same data language Tapestry already uses for its scripting. 

The Godot side of the bridge is [here](https://git.sr.ht/~ionous/tapestry/tree/dev/item/engines/godot/ext/src/tapext.cpp). It takes two Godot strings, converts them to go strings, and calls a function `Post()` written in go. It expects a c-style ( also containing json ) in response, and turns that into a Godot variant ( of dictionaries, arrays, and primitive types. )


```cpp
Variant Tapestry::post( const String& endpoint, const String& json ) {
	// convert the first string:
	CharString endChars = endpoint.utf8(); // copies
	GoString endGo = { endChars.ptr(), endChars.length() };
	// convert the second string:
	CharString jsChars = json.utf8();   // copies
	GoString jsGo = { jsChars.ptr(), jsChars.length() };
	// call our go-function
	const char * result = Post(endGo, jsGo);
	// interpret the response as json, and convert to a variant:
	return JSON::parse_string(result);
}
```

The big gotcha is memory management. I don't know for certain whether Godot allocated memory and Golang allocated memory are pulling from the same heap. If those are different, having one side free memory allocated by the other side would be bad <sup>:tm:</sup>. The setup i chose avoids that issue.

Since `Post()` needs to allocate a string to return it, i also let `Post()` free that string on the next call ( see below. ) The `result` memory, therefore, stays valid between calls. That's more than enough time because `JSON::parse_string( result )` actually copies the string anyway. ( Multiple times, unfortunately. ) And therefore we don't even need the memory after returning from `Tapestry::post()`.


The Golang side of the bridge -- its implementation of `Post()` -- is [here](https://git.sr.ht/~ionous/tapestry/tree/dev/item/engines/godot/ext/src/taplib.go). It looks like this: 

```go
package main

// #include <stdlib.h>
import "C"

//export Post
func Post(endpoint, msg string) (ret *C.char) {
	res, e := post(endpoint, msg)
	if e != nil {
		res = e.Error() // todo: change errors to json: ex. `{"err":...}`
	}
	// free memory from any prior result
	if result != nil {
		C.free(result)
	}
	// create memory for this new result
	ret = C.CString(res)
	result = unsafe.Pointer(ret)
	return
}

func post(endpoint, msg string) (ret string, err error) {
	// in case anything goes wrong, don't crash godot.
	defer func() {
		if r := recover(); r != nil {
			err = fmt.Errorf("Recovered %s:\n%s", r, debug.Stack())
		}
	}()

	// ... CODE CALLING TAPESTRY WITH MSG AND ENDPOINT
	return
}

func main() {
}

// this is the same memory as `char * result` on the godot side.
var result unsafe.Pointer
```

The official `cgo` docs get into the details, but the notable bits are:

1. Uses `package main` with a `main()` function ( which can be empty. )
2. Must `import "C"`, and use include comments to refer to any c functions it needs. ( especially `// #include <stdlib.h>` )
3. Must use export comments to indicate which functions are exposed to godot. ( ex. `//export Post` gives `Post` extern c linkage, making it callable by godot. )
5. Must handle strings and other memory as per the `cgo` docs.
4. Should use a "recover" to catch any panics ( otherwise panics crash godot. )


All in all, though, pretty straight forward.


#### Compiling the go code

To compile the golang side of the bridge, on Windows i used [mingw](https://en.wikipedia.org/wiki/Mingw-w64) via [tdm-gcc](https://jmeubank.github.io/tdm-gcc/). ( It should be possible to use the msvc toolchain as well. )  On MacOS, if you have xcode or its command line compiler, that's all you need.

Once those tools are installed, all that's necessary is:

```
> go build -o taplib.a -buildmode=c-archive taplib.go
```


The `c-archive` mode tells it to make static lib ( the `taplib.a` ). The other option is a `c-shared` dll. Since the godot extension is already a dll, a static lib is a better choice than two dlls.


#### Compiling the extension 

Godot uses `scons` to build. Godot's extension instructions comes with an [SConstruct](https://docs.godotengine.org/en/stable/_downloads/45a3f5e351266601b5e7663dc077fe12/SConstruct) makefile which needs to be modified to include the cgo archive.  This was the trickiest bit because i don't know scons.

At the simplest, it needs the the manually built c-archive added as a dependency:

```python
#!/usr/bin/env python
import os
import sys

env = SConscript("godot-cpp/SConstruct")
env.Append(LIBS=File('src/taplib.a'))  # <--- added this 
env.Append(CPPPATH=["src/"])
sources = Glob("src/*.cpp")

if env["platform"] == "macos":
    library = env.SharedLibrary(
        "demo/bin/libgdexample.{}.{}.framework/libgdexample.{}.{}".format(
            env["platform"], env["target"], env["platform"], env["target"]
        ),
        source=sources,
    )
else:
    library = env.SharedLibrary(
        "demo/bin/libgdexample{}{}".format(env["suffix"], env["SHLIBSUFFIX"]),
        source=sources,
    )
    
Default(library)
```

I also added some instructions to build the go code automatically. To be correct, it would need to use `go list` to detect stale dependencies. ( See "Possible Improvements" below. ) The complete SConstruct is [here](https://git.sr.ht/~ionous/tapestry/tree/dev/item/engines/godot/ext/SConstruct).

Then you need to run scons. I used the mingw option to match the go compiled side.


```
# windows:
> scons use_mingw=true

# macos
> scons arch=x86_64
```



On Windows: installing the [vcredist](https://learn.microsoft.com/en-us/cpp/windows/latest-supported-vc-redist) might be necessary; i initially had some problems loading the extension without that. On MacOS: i had to get the latest [command line tools](https://developer.apple.com/xcode/resources/).


#### Using the extension

The `bin` directory of the godot project needs the compiled extension and a "manifest". The one for Tapestry is [here](https://git.sr.ht/~ionous/tapestry/tree/dev/item/engines/godot/demo/bin/tapestry.gdextension). It looks like:


```ini
# tapestry.gdextension
[configuration]
entry_symbol = "tapestry_library_init"
compatibility_minimum = 4.1

[libraries]
macos.debug = "res://bin/libtapestry.macos.template_debug.framework"
macos.release = "res://bin/libtapestry.macos.template_release.framework"
windows.debug.x86_64 = "res://bin/libtapestry.windows.template_debug.x86_64.dll"
windows.release.x86_64 = "res://bin/libtapestry.windows.template_release.x86_64.dll"
```

That's it. The extension appears in godot as a global class, with the name as it appeared in the extension `.cpp`.

```py
# send a json-friendly variant to tapestry, and get one in return
func _post(endpoint: String, msg: Variant) -> Variant:
  return Tapestry.post(endpoint, JSON.stringify(msg))
```


Possible Improvements:
-----

* Move the command line flags into the scons script ( `arch=x86_64` for osx, `use_mingw=true` for windows )
* Use `go list` in the scons script to determine when to trigger `go build`
* Build a universal macos extension ( requires building both architectures and packaging the results )

