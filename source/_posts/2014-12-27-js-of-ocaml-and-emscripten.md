---
layout: post
title: Js_of_ocaml和Emscripten
author: MaskRay
tags: [ocaml, emscripten]
---

## 在线编译器

使用了Js\_of\_ocaml和emscripten这两个工具把Caml Featherweight移植到了浏览器里：<http://maskray.me/portfolio/caml-featherweight/>。

<!-- more -->

Caml Featherweight分为编译器(OCaml)和解释器(C)两部分，Js\_of\_ocaml可以把其中的OCaml部分编译成JavaScript，emscripten则可以把C编译成JavaScript，这样就可以在浏览器客户端里实现编译和执行。

### Js\_of\_ocaml

如果使用Arch Linux的话，建议不要使用extra/ocaml-4.02.1-1，最好安装opam并用`opam switch 4.02.1`编译一个ocaml。因为extra/camlp4-4.02.0+1-1接口与ocaml不一致，而且不提供/usr/lib/ocaml/camlp4/META，findlib找不到这个库，用这个ocaml编译Js\_of\_ocaml会出错。

Caml Featherweight在编译生成phrase index和链接过程中均需要使用`seek_out`来定位，而Js\_of\_ocaml没有实现这一功能，因此需要做些变更：

-   Js\_of\_ocaml会把安装目录中的`runtime.js`与字节码转成的JavaScript拼接在一起，运行时里实现了库函数，而其中的`seek_out`虽改变了`channel`的`offset`，而控制输出字符的`caml_ml_output`函数并没有利用`offset`信息。

-   Js\_of\_ocaml得到的JavaScript把OCaml标准输出和标准出错里的数据用`console.log`和`console.error`输出到JavaScript终端里，而我们希望数据能显示在HTML的一个编辑器框里，因此需要进行重定向。方法是修改`js_print_stdout`和`js_print_stderr`的实现。

``` {.diff}
--- /home/ray/.opam/4.02.1/lib/js_of_ocaml/runtime.js   2014-12-27 19:02:52.000000000 +0800
+++ /home/ray/tmp/runtime.js    2014-12-27 19:02:10.331759148 +0800
@@ -3890,13 +3890,26 @@
         caml_blit_string(buffer,offset,string,0,len);
     }
     var jsstring = string.toString();
-    var id = jsstring.lastIndexOf("\n");
-    if(id < 0)
-        oc.buffer+=jsstring;
-    else {
+    if (oc.fd <= 2) {
+      var id = jsstring.lastIndexOf("\n");
+      if (id < 0)
+        oc.buffer += jsstring;
+      else {
         oc.buffer+=jsstring.substr(0,id+1);
         caml_ml_flush (oc);
         oc.buffer += jsstring.substr(id+1);
+      }
+    } else {
+      var clen = caml_ml_string_length(oc.file.data);
+      if (oc.offset + len <= clen)
+        caml_blit_string(buffer, 0, oc.file.data, oc.offset, len);
+      else {
+        var new_data = caml_create_string(oc.offset+len);
+        caml_blit_string(oc.file.data, 0, new_data, 0, clen);
+        caml_blit_string(buffer, 0, new_data, oc.offset, len);
+        oc.file.data = new_data;
+      }
+      oc.offset += len;
     }
     return 0;
 }
@@ -4313,19 +4326,13 @@
 function js_print_stdout(s) {
   // Do not output the last \n if present
   // as console logging display a newline at the end
-  if(s.charCodeAt(s.length - 1) == 10)
-    s = s.substr(0,s.length - 1 );
-  var v = joo_global_object.console;
-  v  && v.log && v.log(s);
+  caml_stdout += s;
 }
 //Provides: js_print_stderr
 function js_print_stderr(s) {
   // Do not output the last \n if present
   // as console logging display a newline at the end
-  if(s.charCodeAt(s.length - 1) == 10)
-    s = s.substr(0,s.length - 1 );
-  var v = joo_global_object.console;
-  v && v.error && v.error(s);
+  caml_stdout += s;
 }
 //# 1 "jslib_js_of_ocaml.js"
 // Js_of_ocaml library
```

修改`_tags`文件：

    <*.ml> or "js_main.byte": package(js_of_ocaml), package(js_of_ocaml.syntax), syntax(camlp4o)

编译并翻译成JavaScript：

```bash
cd src
ocamlbuild -use-ocamlfind js_main.byte && js_of_ocaml --no-inline --pretty js_main.byte -o ../build/js/camlfwc.js
```

Js\_of\_ocaml还有些问题，比如`Lexing.from_string`似乎不能用，浮点数小数点的解析也会出错。

### Emscripten

把C/C++编译成JavaScript的项目还有几个，比较成熟的还有[Cheerp](http://leaningtech.com/cheerp/)，但构建似乎不太容易。

Emscripten项目多了一个部件——emscripten-fastcomp，似乎是为LLVM添加了一个新target，但需要编译LLVM，且目前只支持到llvm 3.4，而Arch Linux里的llvm版本号已经是3.5了。Emscripten旧的后端实际上依然可用，`git clone https://github.com/kripken/emscripten`后无需编译直接可用，但很多库函数如`memcmp`、`getopt`等均不支持，因此需要修改解释器去除这部分的依赖。

``` {.bash}
export EMCC_FAST_COMPILER=0
~/tmp/emscripten/emcc -D__WORDSIZE=32 -I src/runtime src/runtime/*.c -o build/js/camlfwrun.js -w
```

### 前端

仿照<http://batsh.org/>，使用Bootstrap 3创建了一个页面，主体是<http://ace.c9.io/>的两个编辑器框，另外放了几个菜单用于加载样例和运行等。

核心部分是下面的LiveScript代码：

```livescript
stdout = ''
stderr = ''
window.caml_stdout = ''
reset = !->
  stdout := ''
  stderr := ''
  window.caml_stdout = ''
window.Module =
  noInitialRun: true
  noExitRuntime: true
  print: (data)!->
    stdout += data+'\n'
run = !->
  #$.post '/run', source.getValue(), (data)!->
  #    if data.err?
  #      output.setValue data.err
  #    else
  #      output.setValue data.output
  #    output.navigateFileStart()
  reset()
  exefile = compile source.getValue()
  if not exefile?
    output.setValue window.caml_stdout
  else
    buf = new Uint8Array exefile.c
    FS.writeFile \a.out, buf, encoding: \binary
    Module.callMain [\a.out]
    if stderr.length > 0
      output.setValue stderr
    else
      output.setValue stdout.replace(/\n$/, '')
  output.navigateFileStart()
pretty = !->
  reset()
  window.pretty source.getValue()
  output.setValue window.caml_stdout
  output.navigateFileStart()
example = (name)!->
  $.get "examples/#{name}.ml", (data)!->
    source
      ..setValue data
      ..navigateFileStart()
source = ace.edit 'source'
  ..setTheme 'ace/theme/tomorrow_night'
  ..setShowPrintMargin false
  ..getSession().setMode 'ace/mode/ocaml'
  ..commands.addCommand do
    name: 'run'
    bindKey: 'Ctrl-Enter'
    exec: run
  ..setOptions do
      fontSize: '18px'
output = ace.edit 'output'
  ..setTheme 'ace/theme/tomorrow_night'
  ..setShowPrintMargin false
  ..setReadOnly true
  ..setHighlightActiveLine false
  ..setOptions do
      fontSize: '18px'
$('#run').on 'click', run
$('#pretty').on 'click', pretty
$('a[data-example]').on 'click', ->
  example $(@).data 'example'
```
