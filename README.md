# Self producing code in HTML with `document.write`

## How does it work?

HTML root usually consists of `head` and `body`. When HTML parser parses script tag (i.e.,`<script>...</script>`), it will pause and let the javascript parser and executor to parse and execute the script. For inline javascript, the javascript parser will look for the sequence `</script>`, and terminate parsing. Whatever after that are assumed to be HTML code rather than javascript code.

```html
<html>
<head>...</head>
<body>
    <script>
        ...
    </script>
</body>
</html>
```

In javascript, there is a specific API called `document.write(...)`. This API allows HTML code to be written after the current script tag. After the HTML code is written, they are immediately parsed and executed before `document.write(...)` returns. 

Since thiis works for any HTML code, it works for script tag as well. So one can write the following code to use javascript to create a script tag, and execute that script.

```html
...
<script>
    document.write("<script>...</script" + ">")
</script>
...
```

Note that we can't write the sequence `</script>` in inline javascript as it will terminate javascript parsing. So we need to use some trick to evaluate a string to have value `</script>` but not write it out in literal form.

Going one step further, if we can create a script that create another script tag with itself, we can create an infinite loop!

```html
...
<script>
    document.write("<script>document.write(...);</script" + ">")
</script>
...
```

From [self-producing programs][self-producing], we know how to write such program. The idea is to create a variable (let's say `s`) with a string literal, write the logic of the code, and finally output the parts lead up to creating `s`, a representation `s`'s content presumably computed using a function that takes `s` as the argument, and the content of `s`. Then, we copy the code from end of `s` to the end of code, using the same function as we compute the representation of `s`, and paste the result in the definition of `s` in term of string literal.

For Python the function that creates the representation of `s` is `s.__repr__()`. For javascript, we can use `JSON.stringify(s)` to achieve the same goal.

As an example, we first reserve the space for `s` and write the code.
```html
...
<script>
    var s = "";
    ...
    console.log("I am executing");
    ...
    document.write("<script>var s = " + JSON.stringify(s) + s + "</script" + ">");
</script>
...
```
Note that we can incldue arbritrary code between `s` and `document.write`.

Once we are done, we copy the code starts from `;` at the end of `var s = "";` to the end of the program (just before the `</script>` tag). Below is the content we copied. To make our life easier, we paste them into a text file called `tmpfile`

```
;
    ...
    console.log("I am executing");
    ...
    document.write("<script>var s = " + JSON.stringify(s) + s + "</script" + ">");
```

We opened a nodejs REPL, run the following lines to read the content of the file and print out its representation (after calling `JSON.stringify`)
```javascript
res = fs.readFileSync("tmpfile");
console.log(JSON.stringify(res.toString()));
```

Then, we paste them inside `s`, so it became the following and we are done.
```javascript
var s = ";\n    ...\n    console.log(\"I am executing\");\n    ...\n    document.write(\"<script>var s = \" + JSON.stringify(s) + s + \"</script\" + \">\");\n";
...
console.log("I am executing");
...
document.write("<script>var s = " + JSON.stringify(s) + s + "</script" + ">");
```

## What actually happens?
When we run the code by loading the webpage into a browser, the page finishes loading, indicating that infinite loop does not happen. In the inspector, we can see that there is 21 script tags, and this behavior is consistent across Chrome, Firefox, and Safari based browsers. So there is a maximum recursion depth, possibly specified by HTML standard, and it's probably around 21. So we cannot actually make browser stuck in infinite loop.

However, this does not stop us from making the browser very slow. Using the technique mentioned above, instead of creating 1 script tag of itself, we can create as many as we want by calling `document.write` any number of times. We can even use a loop to make our life easier. It turns out that when we write 999 script tags per execution, it will cause Chrome and Safari based browser to stuck with high CPU usage for quite a while. Interestingly, Firefox based browser does not hava this problem, and the HTML parser simply stops executing javascript code after creating a certain number of script tags.


# References
[document.write() reference][document-write]

[self producing program guide][self-producing]

[document-write]: https://developer.mozilla.org/en-US/docs/Web/API/Document/write
[self-producing]: https://github.com/lewisxy/self-producing