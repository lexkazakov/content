---
title: Understanding WebAssembly text format
slug: WebAssembly/Understanding_the_text_format
tags:
  - Functions
  - JavaScript
  - S-expressions
  - WebAssembly
  - calls
  - memory
  - shared address
  - table
  - text format
  - was
  - wasm
---
<p>{{WebAssemblySidebar}}</p>

<p>To enable WebAssembly to be read and edited by humans, there is a textual representation of the wasm binary format. This is an intermediate form designed to be exposed in text editors, browser developer tools, etc. This article explains how that text format works, in terms of the raw syntax, and how it is related to the underlying bytecode it represents — and the wrapper objects representing wasm in JavaScript.</p>

<div class="notecard note">
<p><strong>Note:</strong> This is potentially overkill if you are a web developer who just wants to load a wasm module into a page and use it in your code (see <a href="/en-US/docs/WebAssembly/Using_the_JavaScript_API">Using the WebAssembly JavaScript API</a>), but it is more useful if for example, you want to write wasm modules to optimize the performance of your JavaScript library, or build your own WebAssembly compiler.</p>
</div>

<h2 id="S-expressions">S-expressions</h2>

<p>In both the binary and textual formats, the fundamental unit of code in WebAssembly is a module.  In the text format, a module is represented as one big S-expression.  S-expressions are a very old and very simple textual format for representing trees, and thus we can think of a module as a tree of nodes that describe the module’s structure and its code.  Unlike the Abstract Syntax Tree of a programming language, though, WebAssembly’s tree is pretty flat, mostly consisting of lists of instructions.</p>

<p>First, let’s see what an S-expression looks like.  Each node in the tree goes inside a pair of parentheses — <code>( ... )</code>.  The first label inside the parenthesis tells you what type of node it is, and after that there is a space-separated list of either attributes or child nodes.  So that means the WebAssembly S-expression:</p>

<pre class="brush: wasm">(module (memory 1) (func))</pre>

<p>represents a tree with the root node “module” and two child nodes, a "memory" node with the attribute "1" and a "func" node.  We’ll see shortly what these nodes actually mean.</p>

<h3 id="The_simplest_module">The simplest module</h3>

<p>Let's start with the simplest, shortest possible wasm module.</p>

<pre class="brush: wasm">(module)</pre>

<p>This module is totally empty, but is still a valid module.</p>

<p>If we convert our module to binary now (see <a href="/en-US/docs/WebAssembly/Text_format_to_wasm">Converting WebAssembly text format to wasm</a>), we’ll see just the 8 byte module header described in the <a href="https://webassembly.org/docs/binary-encoding/#high-level-structure">binary format</a>:</p>

<pre class="brush: wasm">0000000: 0061 736d              ; WASM_BINARY_MAGIC
0000004: 0100 0000              ; WASM_BINARY_VERSION</pre>

<h3 id="Adding_functionality_to_your_module">Adding functionality to your module</h3>

<p>Ok, that’s not very interesting, let’s add some executable code to this module.</p>

<p>All code in a webassembly module is grouped into functions, which have the following pseudo-code structure:</p>

<pre class="brush: wasm">( func &lt;signature&gt; &lt;locals&gt; &lt;body&gt; )</pre>

<ul>
 <li>The <strong>signature</strong> declares what the function takes (parameters) and returns (return values).</li>
 <li>The <strong>locals</strong> are like vars in JavaScript, but with explicit types declared.</li>
 <li>The <strong>body</strong> is just a linear list of low-level instructions.</li>
</ul>

<p>So this is similar to functions in other languages, even if it looks different because it is an S-expression.</p>

<h2 id="Signatures_and_parameters">Signatures and parameters</h2>

<p>The signature is a sequence of parameter type declarations followed by a list of return type declarations. It is worth noting here that:</p>

<ul>
 <li>
  The absence of a <code>(result)</code> means the function doesn’t return anything.
 </li>
 <li>In the current iteration, there can be at most 1 return type, but <a href="https://github.com/WebAssembly/spec/blob/master/proposals/multi-value/Overview.md">later this will be relaxed</a> to any number.</li>
</ul>

<p>Each parameter has a type explicitly declared; wasm currently has four available number types (plus reference types; see the {{anch("Reference types")}}) section below):</p>

<ul>
 <li><code>i32</code>: 32-bit integer</li>
 <li><code>i64</code>: 64-bit integer</li>
 <li><code>f32</code>: 32-bit float</li>
 <li><code>f64</code>: 64-bit float</li>
</ul>

<p>A single parameter is written <code>(param i32)</code> and the return type is written <code>(result i32)</code>, hence a binary function that takes two 32-bit integers and returns a 64-bit float would be written like this:</p>

<pre class="brush: wasm">(func (param i32) (param i32) (result f64) ... )</pre>

<p>After the signature, locals are listed with their type, for example <code>(local i32)</code>. Parameters are basically just locals that are initialized with the value of the corresponding argument passed by the caller.</p>

<h2 id="Getting_and_setting_locals_and_parameters">Getting and setting locals and parameters</h2>

<p>Locals/parameters can be read and written by the body of the function with the <code>local.get</code> and <code>local.set</code> instructions.</p>

<p>The <code>local.get</code>/<code>local.set</code> commands refer to the item to be got/set by its numeric index: parameters are referred to first, in order of their declaration, followed by locals in order of their declaration.  So given the following function:</p>

<pre class="brush: wasm">(func (param i32) (param f32) (local f64)
  local.get 0
  local.get 1
  local.get 2)</pre>

<p>The instruction <code>local.get 0</code> would get the i32 parameter, <code>local.get 1</code> would get the f32 parameter, and <code>local.get 2</code> would get the f64 local.</p>

<p>There is another issue here — using numeric indices to refer to items can be confusing and annoying, so the text format allows you to name parameters, locals, and most other items by including a name prefixed by a dollar symbol (<code>$</code>) just before the type declaration.</p>

<p>Thus you could rewrite our previous signature like so:</p>

<pre class="brush: wasm">(func (param $p1 i32) (param $p2 f32) (local $loc f64) …)</pre>

<p>And then could write <code>local.get $p1</code> instead of <code>local.get 0</code>, etc.  (Note that when this text gets converted to binary, though, the binary will contain only the integer.)</p>

<h2 id="Stack_machines">Stack machines</h2>

<p>Before we can write a function body, we have to talk about one more thing: <strong>stack machines</strong>. Although the browser compiles it to something more efficient, wasm execution is defined in terms of a stack machine where the basic idea is that every type of instruction pushes and/or pops a certain number of <code>i32</code>/<code>i64</code>/<code>f32</code>/<code>f64</code> values to/from a stack.</p>

<p>For example, <code>local.get</code> is defined to push the value of the local it read onto the stack, and <code>i32.add</code> pops two <code>i32</code> values (it implicitly grabs the previous two values pushed onto the stack), computes their sum (modulo 2^32) and pushes the resulting i32 value.</p>

<p>When a function is called, it starts with an empty stack which is gradually filled up and emptied as the body’s instructions are executed. So for example, after executing the following function:</p>

<pre class="brush: wasm">(func (param $p i32)
  (result i32)
  local.get $p
  local.get $p
  i32.add)</pre>

<p>The stack contains exactly one <code>i32</code> value — the result of the expression (<code>$p + $p</code>), which is handled by <code>i32.add</code>. The return value of a function is just the final value left on the stack.</p>

<p>The WebAssembly validation rules ensure the stack matches exactly: if you declare a <code>(result f32)</code>, then the stack must contain exactly one <code>f32</code> at the end.  If there is no result type, the stack must be empty.</p>

<h2 id="Our_first_function_body">Our first function body</h2>

<p>As mentioned before, the function body is a list of instructions that are followed as the function is called. Putting this together with what we have already learned, we can finally define a module containing our own simple function:</p>

<pre class="brush: wasm">(module
  (func (param $lhs i32) (param $rhs i32) (result i32)
    local.get $lhs
    local.get $rhs
    i32.add))</pre>

<p>This function gets two parameters, adds them together, and returns the result.</p>

<p>There are a lot more things that can be put inside function bodies, but we will start off simple for now, and you’ll see a lot more examples as you go along. For a full list of the available opcodes, consult the <a href="https://webassembly.github.io/spec/core/exec/index.html">webassembly.org Semantics reference</a>.</p>

<h3 id="Calling_the_function">Calling the function</h3>

<p>Our function won’t do very much on its own — now we need to call it. How do we do that? Like in an ES2015 module, wasm functions must be explicitly exported by an <code>export</code> statement inside the module.</p>

<p>Like locals, functions are identified by an index by default, but for convenience, they can be named. Let's start by doing this — first, we'll add a name preceded by a dollar sign, just after the <code>func</code> keyword:</p>

<pre class="brush: wasm">(func $add … )</pre>

<p>Now we need to add an export declaration — this looks like so:</p>

<pre class="brush: wasm">(export "add" (func $add))</pre>

<p>Here, <code>add</code> is the name the function will be identified by in JavaScript whereas <code>$add</code> picks out which WebAssembly function inside the Module is being exported.</p>

<p>So our final module (for now) looks like this:</p>

<pre class="brush: wasm">(module
  (func $add (param $lhs i32) (param $rhs i32) (result i32)
    local.get $lhs
    local.get $rhs
    i32.add)
  (export "add" (func $add))
)</pre>

<p>If you want to follow along with the example, save the above our module into a file called <code>add.wat</code>, then convert it into a binary file called <code>add.wasm</code> using wabt (see <a href="/en-US/docs/WebAssembly/Text_format_to_wasm">Converting WebAssembly text format to wasm</a> for details).</p>

<p>Next, we’ll load our binary into a typed array called <code>addCode</code> (as described in <a href="/en-US/docs/WebAssembly/Loading_and_running">Fetching WebAssembly Bytecode</a>), compile and instantiate it, and execute our <code>add</code> function in JavaScript (we can now find <code>add()</code> in the <code><a href="/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Instance/exports">exports</a></code> property of the instance):</p>

<pre class="brush: js">WebAssembly.instantiateStreaming(fetch('add.wasm'))
  .then(obj =&gt; {
    console.log(obj.instance.exports.add(1, 2));  // "3"
  });</pre>

<div class="notecard note">
<p><strong>Note:</strong> You can find this example in GitHub as <a href="https://github.com/mdn/webassembly-examples/blob/master/understanding-text-format/add.html">add.html</a> (<a href="https://mdn.github.io/webassembly-examples/understanding-text-format/add.html">see it live also</a>). Also see {{JSxRef("WebAssembly.instantiateStreaming()")}} for more details about the instantiate function.</p>
</div>

<h2 id="Exploring_fundamentals">Exploring fundamentals</h2>

<p>Now we’ve covered the real basics, let’s move on to look at some more advanced features.</p>

<h3 id="Calling_functions_from_other_functions_in_the_same_module">Calling functions from other functions in the same module</h3>

<p>The <code>call</code> instruction calls a single function, given its index or name. For example, the following module contains two functions — one just returns the value 42, the other returns the result of calling the first plus one:</p>

<pre class="brush: wasm">(module
  (func $getAnswer (result i32)
    i32.const 42)
  (func (export "getAnswerPlus1") (result i32)
    call $getAnswer
    i32.const 1
    i32.add))</pre>

<div class="notecard note">
<p><strong>Note:</strong> <code>i32.const</code> just defines a 32-bit integer and pushes it onto the stack. You could swap out the <code>i32</code> for any of the other available types, and change the value of the const to whatever you like (here we’ve set the value to <code>42</code>).</p>
</div>

<p>In this example you’ll notice an <code>(export "getAnswerPlus1")</code> section, declared just after the <code>func</code> statement in the second function — this is a shorthand way of declaring that we want to export this function, and defining the name we want to export it as.</p>

<p>This is functionally equivalent to including a separate function statement outside the function, elsewhere in the module in the same manner as we did before, e.g.:</p>

<pre class="brush: wasm">(export "getAnswerPlus1" (func $functionName))</pre>

<p>The JavaScript code to call our above module looks like so:</p>

<pre class="brush: js">WebAssembly.instantiateStreaming(fetch('call.wasm'))
 .then(obj =&gt; {
    console.log(obj.instance.exports.getAnswerPlus1());  // "43"
  });</pre>

<h3 id="Importing_functions_from_JavaScript">Importing functions from JavaScript</h3>

<p>We have already seen JavaScript calling WebAssembly functions, but what about WebAssembly calling JavaScript functions? WebAssembly doesn’t actually have any built-in knowledge of JavaScript, but it does have a general way to import functions that can accept either JavaScript or wasm functions. Let’s look at an example:</p>

<pre class="brush: wasm">(module
  (import "console" "log" (func $log (param i32)))
  (func (export "logIt")
    i32.const 13
    call $log))</pre>

<p>WebAssembly has a two-level namespace so the import statement here is saying that we’re asking to import the <code>log</code> function from the <code>console</code> module. You can also see that the exported <code>logIt</code> function calls the imported function using the <code>call</code> instruction we introduced above.</p>

<p>Imported functions are just like normal functions: they have a signature that WebAssembly validation checks statically, and they are given an index and can be named and called.</p>

<p>JavaScript functions have no notion of signature, so any JavaScript function can be passed, regardless of the import’s declared signature. Once a module declares an import, the caller of {{JSxRef("WebAssembly.instantiate()")}} must pass in an import object that has the corresponding properties.</p>

<p>For the above, we need an object (let's call it <code>importObject</code>) such that <code>importObject.console.log</code> is a JavaScript function.</p>

<p>This would look like the following:</p>

<pre class="brush: js">var importObject = {
  console: {
    log: function(arg) {
      console.log(arg);
    }
  }
};

WebAssembly.instantiateStreaming(fetch('logger.wasm'), importObject)
  .then(obj =&gt; {
    obj.instance.exports.logIt();
  });</pre>

<div class="notecard note">
<p><strong>Note:</strong> You can find this example on GitHub as <a href="https://github.com/mdn/webassembly-examples/blob/master/understanding-text-format/logger.html">logger.html</a> (<a href="https://mdn.github.io/webassembly-examples/understanding-text-format/logger.html">see it live also</a>).</p>
</div>

<h3 id="Declaring_globals_in_WebAssembly">Declaring globals in WebAssembly</h3>

<p>WebAssembly has the ability to create global variable instances, accessible from both JavaScript and importable/exportable across one or more {{JSxRef("WebAssembly.Module")}} instances. This is very useful, as it allows dynamic linking of multiple modules.</p>

<p>In WebAssembly text format, it looks something like this (see <a href="https://github.com/mdn/webassembly-examples/blob/master/js-api-examples/global.wat">global.wat</a> in our GitHub repo; also see <a href="https://mdn.github.io/webassembly-examples/js-api-examples/global.html">global.html</a> for a live JavaScript example):</p>

<pre class="brush: wasm">(module
   (global $g (import "js" "global") (mut i32))
   (func (export "getGlobal") (result i32)
        (global.get $g))
   (func (export "incGlobal")
        (global.set $g
            (i32.add (global.get $g) (i32.const 1))))
)</pre>

<p>This looks similar to what we've seen before, except that we specify a global value using the keyword <code>global</code>, and we also specify the keyword <code>mut</code> along with the value's datatype if we want it to be mutable.</p>

<p>To create an equivalent value using JavaScript, you'd use the {{JSxRef("WebAssembly.Global()")}} constructor:</p>

<pre class="brush: js">const global = new WebAssembly.Global({value: "i32", mutable: true}, 0);</pre>

<h3 id="WebAssembly_Memory">WebAssembly Memory</h3>

<p>The above example is a pretty terrible logging function: it only prints a single integer!  What if we wanted to log a text string? To deal with strings and other more complex data types, WebAssembly provides <strong>memory</strong> (although we also have {{anch("Reference types")}} in newer implementation of WebAssembly). According to WebAssembly, memory is just a large array of bytes that can grow over time. WebAssembly contains instructions like <code>i32.load</code> and <code>i32.store</code> for reading and writing from <a href="https://webassembly.org/docs/semantics/#linear-memory">linear memory</a>.</p>

<p>From JavaScript’s point of view, it’s as though memory is all inside one big (resizable) {{jsxref("ArrayBuffer")}}. That’s literally all that asm.js has to play with (except that it isn't resizable; see the asm.js <a href="http://asmjs.org/spec/latest/#programming-model">Programming model</a>).</p>

<p>So a string is just a sequence of bytes somewhere inside this linear memory. Let's assume that we’ve written a suitable string of bytes to memory; how do we pass that string out to JavaScript?</p>

<p>The key is that JavaScript can create WebAssembly linear memory instances via the {{JSxRef("WebAssembly.Memory()")}} interface, and access an existing memory instance (currently you can only have one per module instance) using the associated instance methods. Memory instances have a <code><a href="/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Memory/buffer">buffer</a></code> getter, which returns an <code>ArrayBuffer</code> that points at the whole linear memory.</p>

<p>Memory instances can also grow, for example via the <code><a href="/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Memory/grow">Memory.grow()</a></code> method in JavaScript. When growth occurs, since <code>ArrayBuffer</code>s can’t change size, the current <code>ArrayBuffer</code> is detached and a new <code>ArrayBuffer</code> is created to point to the newer, bigger memory. This means all we need to do to pass a string to JavaScript is to pass out the offset of the string in linear memory along with some way to indicate the length.</p>

<p>While there are many different ways to encode a string’s length in the string itself (for example, C strings); for simplicity here we just pass both offset and length as parameters:</p>

<pre class="brush: wasm">(import "console" "log" (func $log (param i32) (param i32)))</pre>

<p>On the JavaScript side, we can use the <a href="/en-US/docs/Web/API/TextDecoder">TextDecoder API</a> to easily decode our bytes into a JavaScript string.  (We specify <code>utf8</code> here, but many other encodings are supported.)</p>

<pre class="brush: js">function consoleLogString(offset, length) {
  var bytes = new Uint8Array(memory.buffer, offset, length);
  var string = new TextDecoder('utf8').decode(bytes);
  console.log(string);
}</pre>

<p>The last missing piece of the puzzle is where <code>consoleLogString</code> gets access to the WebAssembly <code>memory</code>. WebAssembly gives us a lot of flexibility here: we can either create a <code><a href="/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Memory">Memory</a></code> object in JavaScript and have the WebAssembly module import the memory, or we can have the WebAssembly module create the memory and export it to JavaScript.</p>

<p>For simplicity, let's create it in JavaScript then import it into WebAssembly.  Our <code>import</code> statement is written as follows:</p>

<pre class="brush: wasm">(import "js" "mem" (memory 1))</pre>

<p>The <code>1</code> indicates that the imported memory must have at least 1 page of memory (WebAssembly defines a page to be 64KB.)</p>

<p>So let's see a complete module that prints the string “Hi”.  In a normal compiled C program, you’d call a function to allocate some memory for the string, but since we’re just writing our own assembly here and we own the entire linear memory, we can just write the string contents into global memory using a <code>data</code> section.  Data sections allow a string of bytes to be written at a given offset at instantiation time and are similar to the <code>.data</code> sections in native executable formats.</p>

<p>Our final wasm module looks like this:</p>

<pre class="brush: wasm">(module
  (import "console" "log" (func $log (param i32 i32)))
  (import "js" "mem" (memory 1))
  (data (i32.const 0) "Hi")
  (func (export "writeHi")
    i32.const 0  ;; pass offset 0 to log
    i32.const 2  ;; pass length 2 to log
    call $log))</pre>

<div class="notecard note">
<p><strong>Note:</strong> Above, note the double semi-colon syntax (<code>;;</code>) for allowing comments in WebAssembly files.</p>
</div>

<p>Now from JavaScript we can create a Memory with 1 page and pass it in. This results in "Hi" being printed to the console:</p>

<pre class="brush: js">var memory = new WebAssembly.Memory({initial:1});

var importObject = { console: { log: consoleLogString }, js: { mem: memory } };

WebAssembly.instantiateStreaming(fetch('logger2.wasm'), importObject)
  .then(obj =&gt; {
    obj.instance.exports.writeHi();
  });</pre>

<div class="notecard note">
<p><strong>Note:</strong> You can find the full source on GitHub as <a href="https://github.com/mdn/webassembly-examples/blob/master/understanding-text-format/logger2.html">logger2.html</a> (<a href="https://mdn.github.io/webassembly-examples/understanding-text-format/logger2.html">also see it live</a>).</p>
</div>

<h3 id="WebAssembly_tables">WebAssembly tables</h3>

<p>To finish this tour of the WebAssembly text format, let’s look at the most intricate, and often confusing, part of WebAssembly: <strong>tables</strong>. Tables are basically resizable arrays of references that can be accessed by index from WebAssembly code.</p>

<p>To see why tables are needed, we need to first observe that the <code>call</code> instruction we saw earlier (see {{anch("Calling functions from other functions in the same module")}}) takes a static function index and thus can only ever call one function — but what if the callee is a runtime value?</p>

<ul>
 <li>In JavaScript we see this all the time: functions are first-class values.</li>
 <li>In C/C++, we see this with function pointers.</li>
 <li>In C++, we see this with virtual functions.</li>
</ul>

<p>WebAssembly needed a type of call instruction to achieve this, so we gave it <code>call_indirect</code>, which takes a dynamic function operand. The problem is that the only types we have to give operands in WebAssembly are (currently) <code>i32</code>/<code>i64</code>/<code>f32</code>/<code>f64</code>.</p>

<p>WebAssembly could add an <code>anyfunc</code> type ("any" because the type could hold functions of any signature), but unfortunately this <code>anyfunc</code> type couldn’t be stored in linear memory for security reasons. Linear memory exposes the raw contents of stored values as bytes and this would allow wasm content to arbitrarily observe and corrupt raw function addresses, which is something that cannot be allowed on the web.</p>

<p>The solution was to store function references in a table and pass around table indices instead, which are just i32 values. <code>call_indirect</code>’s operand can therefore be an i32 index value.</p>

<h4 id="Defining_a_table_in_wasm">Defining a table in wasm</h4>

<p>So how do we place wasm functions in our table? Just like <code>data</code> sections can be used to initialize regions of linear memory with bytes, <code>elem</code> sections can be used to initialize regions of tables with functions:</p>

<pre class="brush: wasm">(module
  (table 2 funcref)
  (elem (i32.const 0) $f1 $f2)
  (func $f1 (result i32)
    i32.const 42)
  (func $f2 (result i32)
    i32.const 13)
  ...
)</pre>

<ul>
 <li>In <code>(table 2 funcref)</code>, the 2 is the initial size of the table (meaning it will store two references) and <code>funcref</code> declares that the element type of these references are function reference.</li>
 <li>The functions (<code>func</code>) sections are just like any other declared wasm functions. These are the functions we are going to refer to in our table (for example’s sake, each one just returns a constant value). Note that the order the sections are declared in doesn’t matter here — you can declare your functions anywhere and still refer to them in your <code>elem</code> section.</li>
 <li>The <code>elem</code> section can list any subset of the functions in a module, in any order, allowing duplicates. This is a list of the functions that are to be referenced by the table, in the order they are to be referenced.</li>
 <li>The <code>(i32.const 0)</code> value inside the <code>elem</code> section is an offset — this needs to be declared at the start of the section, and specifies at what index in the table function references start to be populated. Here we’ve specified 0, and a size of 2 (see above), so we can fill in two references at indexes 0 and 1. If we wanted to start writing our references at offset 1, we’d have to write <code>(i32.const 1)</code>, and the table size would have to be 3.</li>
</ul>

<div class="notecard note">
<p><strong>Note:</strong> Uninitialized elements are given a default throw-on-call value.</p>
</div>

<p>In JavaScript, the equivalent calls to create such a table instance would look something like this:</p>

<pre class="brush: js">function() {
  // table section
  var tbl = new WebAssembly.Table({initial:2, element:"funcref"});

  // function sections:
  var f1 = ... /* some imported WebAssembly function */
  var f2 = ... /* some imported WebAssembly function */

  // elem section
  tbl.set(0, f1);
  tbl.set(1, f2);
};</pre>

<h4 id="Using_the_table">Using the table</h4>

<p>Moving on, now we’ve defined the table we need to use it somehow. Let's use this section of code to do so:</p>

<pre class="brush: wasm">(type $return_i32 (func (result i32))) ;; if this was f32, type checking would fail
(func (export "callByIndex") (param $i i32) (result i32)
  local.get $i
  call_indirect (type $return_i32))</pre>

<ul>
 <li>The <code>(type $return_i32 (func (result i32)))</code> block specifies a type, with a reference name. This type is used when performing type checking of the table function reference calls later on. Here we are saying that the references need to be functions that return an <code>i32</code> as a result.</li>
 <li>Next, we define a function that will be exported with the name <code>callByIndex</code>. This will take one <code>i32</code> as a parameter, which is given the argument name <code>$i</code>.</li>
 <li>Inside the function, we add one value to the stack — whatever value is passed in as the parameter <code>$i</code>.</li>
 <li>Finally, we use <code>call_indirect</code> to call a function from the table — it implicitly pops the value of <code>$i</code> off the stack. The net result of this is that the <code>callByIndex</code> function invokes the <code>$i</code>’th function in the table.</li>
</ul>

<p>You could also declare the <code>call_indirect</code> parameter explicitly during the command call instead of before it, like this:</p>

<pre class="brush: wasm">(call_indirect (type $return_i32) (local.get $i))</pre>

<p>In a higher level, more expressive language like JavaScript, you could imagine doing the same thing with an array (or probably more likely, object) containing functions. The pseudo code would look something like <code>tbl[i]()</code>.</p>

<p>So, back to the typechecking. Since WebAssembly is typechecked, and the <code>funcref</code> can be potentially any function signature, we have to supply the presumed signature of the callee at the callsite, hence we include the <code>$return_i32</code> type, to tell the program a function returning an <code>i32</code> is expected. If the callee doesn’t have a matching signature (say an <code>f32</code> is returned instead), a {{JSxRef("WebAssembly.RuntimeError")}} is thrown.</p>

<p>So what links the <code>call_indirect</code> to the table we are calling? The answer is that there is only one table allowed right now per module instance, and that is what <code>call_indirect</code> is implicitly calling. In the future, when multiple tables are allowed, we would also need to specify a table identifier of some kind, along the lines of</p>

<pre class="brush: wasm">call_indirect $my_spicy_table (type $i32_to_void)</pre>

<p>The full module all together looks like this, and can be found in our <a href="https://github.com/mdn/webassembly-examples/blob/master/understanding-text-format/wasm-table.wat">wasm-table.wat</a> example file:</p>

<pre class="brush: wasm">(module
  (table 2 funcref)
  (func $f1 (result i32)
    i32.const 42)
  (func $f2 (result i32)
    i32.const 13)
  (elem (i32.const 0) $f1 $f2)
  (type $return_i32 (func (result i32)))
  (func (export "callByIndex") (param $i i32) (result i32)
    local.get $i
    call_indirect (type $return_i32))
)</pre>

<p>We load it into a webpage using the following JavaScript:</p>

<pre class="brush: js">WebAssembly.instantiateStreaming(fetch('wasm-table.wasm'))
  .then(obj =&gt; {
    console.log(obj.instance.exports.callByIndex(0)); // returns 42
    console.log(obj.instance.exports.callByIndex(1)); // returns 13
    console.log(obj.instance.exports.callByIndex(2)); // returns an error, because there is no index position 2 in the table
  });</pre>

<div class="notecard note">
<p><strong>Note:</strong> You can find this example on GitHub as <a href="https://github.com/mdn/webassembly-examples/blob/master/understanding-text-format/wasm-table.html">wasm-table.html</a> (<a href="https://mdn.github.io/webassembly-examples/understanding-text-format/wasm-table.html">see it live also</a>).</p>
</div>

<div class="notecard note">
<p><strong>Note:</strong> Just like Memory, Tables can also be created from JavaScript (see <code><a href="/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Table">WebAssembly.Table()</a></code>) as well as imported to/from another wasm module.</p>
</div>

<h3 id="Mutating_tables_and_dynamic_linking">Mutating tables and dynamic linking</h3>

<p>Because JavaScript has full access to function references, the Table object can be mutated from JavaScript using the <code><a href="/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Table/grow">grow()</a></code>, <code><a href="/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Table/get">get()</a></code> and <code><a href="/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Table/set">set()</a></code> methods. And WebAssembly code is itself able to manipulate tables using instructions added as part of {{anch("Reference types")}}, such as <code>table.get</code> and <code>table.set</code>.</p>

<p>Because tables are mutable, they can be used to implement sophisticated load-time and run-time <a href="https://webassembly.org/docs/dynamic-linking">dynamic linking schemes</a>. When a program is dynamically linked, multiple instances share the same memory and table. This is symmetric to a native application where multiple compiled <code>.dll</code>s share a single process’s address space.</p>

<p>To see this in action, we’ll create a single import object containing a Memory object and a Table object, and pass this same import object to multiple <code><a href="/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/instantiate">instantiate()</a></code> calls.</p>

<p>Our <code>.wat</code> examples look like so:</p>

<p><code>shared0.wat</code>:</p>

<pre class="brush: wasm">(module
  (import "js" "memory" (memory 1))
  (import "js" "table" (table 1 funcref))
  (elem (i32.const 0) $shared0func)
  (func $shared0func (result i32)
   i32.const 0
   i32.load)
)</pre>

<p><code>shared1.wat</code>:</p>

<pre class="brush: wasm">(module
  (import "js" "memory" (memory 1))
  (import "js" "table" (table 1 funcref))
  (type $void_to_i32 (func (result i32)))
  (func (export "doIt") (result i32)
   i32.const 0
   i32.const 42
   i32.store  ;; store 42 at address 0
   i32.const 0
   call_indirect (type $void_to_i32))
)</pre>

<p>These work as follows:</p>

<ol>
 <li>The function <code>shared0func</code> is defined in <code>shared0.wat</code>, and stored in our imported table.</li>
 <li>This function creates a constant containing the value <code>0</code>, and then uses the <code>i32.load</code> command to load the value contained in the provided memory index. The index provided is <code>0</code> — again, it implicitly pops the previous value off the stack. So <code>shared0func</code> loads and returns the value stored at memory index <code>0</code>.</li>
 <li>In <code>shared1.wat</code>, we export a function called <code>doIt</code> — this function creates two constants containing the values <code>0</code> and <code>42</code>, then calls <code>i32.store</code> to store a provided value at a provided index of the imported memory. Again, it implicitly pops these values off the stack, so the result is that it stores the value <code>42</code> in memory index <code>0</code>,</li>
 <li>In the last part of the function, we create a constant with value <code>0</code>, then call the function at this index 0 of the table, which is <code>shared0func</code>, stored there earlier by the <code>elem</code> block in <code>shared0.wat</code>.</li>
 <li>When called, <code>shared0func</code> loads the <code>42</code> we stored in memory using the <code>i32.store</code> command in <code>shared1.wat</code>.</li>
</ol>

<div class="notecard note">
<p><strong>Note:</strong> The above expressions again pop values from the stack implicitly, but you could declare these explicitly inside the command calls instead, for example:</p>

<pre class="brush: wasm">(i32.store (i32.const 0) (i32.const 42))
(call_indirect (type $void_to_i32) (i32.const 0))</pre>

</div>

<p>After converting to assembly, we then use <code>shared0.wasm</code> and <code>shared1.wasm</code> in JavaScript via the following code:</p>

<pre class="brush: js">var importObj = {
  js: {
    memory : new WebAssembly.Memory({ initial: 1 }),
    table : new WebAssembly.Table({ initial: 1, element: "funcref" })
  }
};

Promise.all([
  WebAssembly.instantiateStreaming(fetch('shared0.wasm'), importObj),
  WebAssembly.instantiateStreaming(fetch('shared1.wasm'), importObj)
]).then(function(results) {
  console.log(results[1].instance.exports.doIt());  // prints 42
});</pre>

<p>Each of the modules that is being compiled can import the same memory and table objects and thus share the same linear memory and table "address space".</p>

<div class="notecard note">
<p><strong>Note:</strong> You can find this example on GitHub as <a href="https://github.com/mdn/webassembly-examples/blob/master/understanding-text-format/shared-address-space.html">shared-address-space.html</a> (<a href="https://mdn.github.io/webassembly-examples/understanding-text-format/shared-address-space.html">see it live also</a>).</p>
</div>

<h2 id="Bulk_memory_operations">Bulk memory operations</h2>

<p>Bulk memory operations are a newer addition to the language (for example, in <a href="/en-US/docs/Mozilla/Firefox/Releases/79">Firefox 79</a>) — seven new built-in operations are provided for bulk memory operations such as copying and initializing, to allow WebAssembly to model native functions such as <code>memcpy</code> and <code>memmove</code> in a more efficient, performant way.</p>

<p>The new operations are:</p>

<ul>
 <li><code>data.drop</code>: Discard the data in an data segment.</li>
 <li><code>elem.drop</code>: Discard the data in an element segment.</li>
 <li><code>memory.copy</code>: Copy from one region of linear memory to another.</li>
 <li><code>memory.fill</code>: Fill a region of linear memory with a given byte value.</li>
 <li><code>memory.init</code>: Copy a region from a data segment.</li>
 <li><code>table.copy</code>: Copy from one region of a table to another.</li>
 <li><code>table.init</code>: Copy a region from an element segment.</li>
</ul>

<div class="notecard note">
<p><strong>Note:</strong> You can find more information in the <a href="https://github.com/WebAssembly/bulk-memory-operations/blob/master/proposals/bulk-memory-operations/Overview.md">Bulk Memory Operations and Conditional Segment Initialization</a> proposal.</p>
</div>

<h2 id="Reference_types">Reference types</h2>

<p>The <a href="https://github.com/WebAssembly/reference-types/blob/master/proposals/reference-types/Overview.md">reference types proposal</a> (supported in <a href="/en-US/docs/Mozilla/Firefox/Releases/79">Firefox 79</a>) provides two main features:</p>

<ul>
 <li>A new type, <code>externref</code>, which can hold <em>any</em> JavaScript value, for example strings, DOM references, objects, etc. <code>externref</code> is opaque from the point of view of WebAssembly — a wasm module can't access and manipulate these values and instead can only receive them and pass them back out. But this is very useful for allowing wasm modules to call JavaScript functions, DOM APIs, etc., and generally to pave the way for easier interoperability with the host environment. <code>externref</code> can be used for value types and table elements.</li>
 <li>A number of new instructions that allow wasm modules to directly manipulate {{anch("WebAssembly tables")}}, rather than having to do it via the JavaScript API.</li>
</ul>

<div class="notecard note">
<p><strong>Note:</strong> The <a href="https://rustwasm.github.io/docs/wasm-bindgen/">wasm-bindgen</a> documentation contains some useful information on how to take advantage of <code>externref</code> from Rust.</p>
</div>

<h2 id="Multi-value_WebAssembly">Multi-value WebAssembly</h2>

<p>Another more recent addition to the language (for example, in <a href="/en-US/docs/Mozilla/Firefox/Releases/78">Firefox 78</a>) is WebAssembly multi-value, meaning that WebAssembly functions can now return multiple values, and instruction sequences can consume and produce multiple stack values.</p>

<p>At the time of writing (June 2020) this is at an early stage, and the only multi-value instructions available are calls to functions that themselves return multiple values. For example:</p>

<pre class="brush: wasm">(module
  (func $get_two_numbers (result i32 i32)
    i32.const 1
    i32.const 2
  )
  (func (export "add_to_numbers") (result i32)
    call $get_two_numbers
    i32.add
  )
)</pre>

<p>But this will pave the way for more useful instruction types, and other things besides. For a useful write up of progress so far and how this works, see <a href="https://hacks.mozilla.org/2019/11/multi-value-all-the-wasm/">Multi-Value All The Wasm!</a> by Nick Fitzgerald.</p>

<h2 id="WebAssembly_threads">WebAssembly threads</h2>

<p>WebAssembly Threads (supported in <a href="/en-US/docs/Mozilla/Firefox/Releases/79">Firefox 79</a> onwards) allow WebAssembly Memory objects to be shared across multiple WebAssembly instances running in separate Web Workers, in the same fashion as <code><a href="/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer">SharedArrayBuffer</a></code>s in JavaScript. This allows very fast communication between Workers, and significant performance gains in web applications.</p>

<p>The threads proposal has two parts, shared memories and atomic memory accesses.</p>

<h3 id="Shared_memories">Shared memories</h3>

<p>As described above, you can create shared WebAssembly <code><a href="/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Memory">Memory</a></code> objects, which can be transferred between Window and Worker contexts using <code><a href="/en-US/docs/Web/API/Window/postMessage">postMessage()</a></code>, in the same fashion as a <code><a href="/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer">SharedArrayBuffer</a></code>.</p>

<p>Over on the JavaScript API side, the <code><a href="/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Memory/Memory">WebAssembly.Memory()</a></code> constructor's initialization object now has a <code>shared</code> property, which when set to <code>true</code> will create a shared memory:</p>

<pre class="brush: js">let memory = new WebAssembly.Memory({initial:10, maximum:100, shared:true});
</pre>

<p>the memory's <code><a href="/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Memory/buffer">buffer</a></code> property will now return a <code>SharedArrayBuffer</code>, instead of the usual <code>ArrayBuffer</code>:</p>

<pre class="brush: js">memory.buffer // returns SharedArrayBuffer
</pre>

<p>Over in the text format, you can create a shared memory using the <code>shared</code> keyword, like this:</p>

<pre class="brush: wasm">(memory 1 2 shared)</pre>

<p>Unlike unshared memories, shared memories must specify a "maximum" size, in both the JavaScript API constructor and wasm text format.</p>

<div class="notecard note">
<p><strong>Note:</strong> You can find a lot more details in the <a href="https://github.com/WebAssembly/threads/blob/master/proposals/threads/Overview.md">Threading proposal for WebAssembly</a>.</p>
</div>

<h3 id="Atomic_memory_accesses">Atomic memory accesses</h3>

<p>A number of new wasm instructions have been added that can be used to implement higher level features like mutexes, condition variables, etc. You can <a href="https://github.com/WebAssembly/threads/blob/master/proposals/threads/Overview.md#atomic-memory-accesses">find them listed here</a>. These instructions are allowed on non-shared memories as of Firefox 80.</p>

<div class="notecard note">
<p><strong>Note:</strong> The <a href="https://emscripten.org/docs/porting/pthreads.html">Emscripten Pthreads support page</a> shows how to take advantage of this new functionality from Emscripten.</p>
</div>

<h2 id="Summary">Summary</h2>

<p>This finishes our high-level tour of the major components of the WebAssembly text format and how they get reflected in the WebAssembly JS API.</p>

<h2 id="See_also">See also</h2>

<ul>
 <li>The main thing that wasn’t included is a comprehensive list of all the instructions that can occur in function bodies.  See the <a href="https://webassembly.github.io/spec/core/exec/index.html">WebAssembly semantics</a> for a treatment of each instruction.</li>
 <li>See also the <a href="https://github.com/WebAssembly/spec/blob/master/interpreter/README.md#s-expression-syntax">grammar of the text format</a> that is implemented by the spec interpreter.</li>
</ul>