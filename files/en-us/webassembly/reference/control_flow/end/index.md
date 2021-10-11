---
title: end
slug: WebAssembly/Reference/Control_flow/end
tags:
- WebAssembly
- wasm
- Landing page
- Reference
- Control flow
---
<p>{{WebAssemblySidebar}}</p>

<p><span class="seoSummary"><strong><code>end</code></strong> is used to end a <code>block</code>, <code>loop</code>, <code>if</code>, or <code>else</code>. In the other examples we used the s-expression syntax which doesn't require the <code>end</code>, so you won't find it in the other examples here. However, it's still useful to know about since this is what the browsers display in devtools.</span></p>

<h2 id="Syntax">Syntax</h2>

<pre class="brush: wasm">
i32.const 0
if
  ;; do something
end
</pre>


<h2 id="Full_working_example">Full working example</h2>

<p>Wasm file</p>

<pre class="brush: wasm">
(module
  ;; import the browser console object, you'll need to pass this in from JavaScript
  (import "console" "log" (func $log (param i32)))

  (func
    i32.const 0 ;; change to positive number if you want to run the if block
    if
      i32.const 1
      call $log ;; should log '1'
    end
  )

  (start 1) ;; run the first function automatically
)
</pre>

<p>JavaScript file</p>

<pre class="brush: js">
WebAssembly.instantiateStreaming(
  fetch("link to .wasm file"),
  { console }
);
</pre>

<table>
 <thead>
  <tr>
   <th>Instruction</th>
   <th>Binary opcode</th>
  </tr>
 </thead>
 <tbody>
  <tr>
    <td><code>end</code></td>
    <td><code>0x0b</code></td>
  </tr>
 </tbody>
</table>