### Remarks

Programs were written in 80 characters wide.
These days, this limit has been relaxed because of larger screens.
[Rubocop's default limit is now also 120 characters.][rubocop]

[rubocop]: https://github.com/rubocop/rubocop/pull/7952

However, I do not favor this trend. We live in a mobile-first age.
I often review code on mobile devices.
It makes me sick to see Ruby code that didn't fit on my mobile screen.
Even 80 characters is too long for me.

Therefore, I developed a novel tool to make Ruby code as narrow as possible.
The output code would never overflow horizontally on any mobile device.

### Description

This program converts Ruby code to one whose (almost) every line is two-character long.
Try this:

```
$ cat hello.rb
puts "Hello, TRICK judges!"

$ ruby entry.rb hello.rb
```

You can run the output as follows:

```
$ ruby entry.rb hello.rb > hello2.rb

$ ruby hello2.rb
Hello, world!
```

Of course, (almost) every line of the program itself is two-character long.

```
$ cat entry.rb
```

### Internals

Do not read the following if you want to read the program yourself.

.
.
.
.
.
.
.
.
.
.

Here is the basic structure of an output program.

```
-> &b { b["dummy", "<input program>"] }[&:eval]
```

This creates a proc of `Kernel#eval` by exploiting implicit conversion of `#to_proc` and applies it to the string of the original program.

Character literals are exploited to construct a string.

```
a = ?p
b = ?u
c = ?t
d = ?s
a + b + c + d  #=> "puts"
```

The most difficult part for me was to create a Symbol of `:eval`.
The basic idea to achieve this is abusing a String range:

```
(:a..:zzzz).to_a[102789]  #=> :eval
```

Note that we need to avoid `:zzzz` too.
To managet this difficulty, I read the current iteration algorithm of a String range:

* It iterates only when the beginning value is smaller than the end value
* It applies `String#succ` to the beginning value repeatedly and stops the iteration when it becomes "longer (in byte)" than the end value.

After a whole day of contemplation, I finally found an ideal end value: `:ÀÀ`.
It is two-character long on modern editors, while it has four bytes.

```
(:a..:ÀÀ).to_a[102789] #=> :eval

# Or:
[*:a..:ÀÀ][3**7 * 47] #=> :eval
```

A careful reader would notice that `:ÀÀ` is three-character long.
This can be worked around by using a backslash.

```
[*
:a
..
:\
ÀÀ
][
3\
**
7*
47
]
```

### Limitation

Unfortunately, only the first line has three-characters long. (I said "ALMOST every line"!)

```
->\
```

I don't see why a newline is not allowed after `->`.
It is possible to allow it by applying the following patch to the ruby interpreter's parser with no parsing conflicts.

```
--- parse.y
+++ parse.y
@@ -3788,7 +3788,10 @@ bvar		: tIDENTIFIER
 		    }
 		;
 
-lambda		: tLAMBDA
+lambda_header   : tLAMBDA opt_nl
+                ;
+
+lambda		: lambda_header
 		    {
 			token_info_push(p, "->", &@1);
 			$<vars>1 = dyna_push(p);
```

But I don't know if Matz would accept this.
I'd like to leave the true two-character Ruby style under the current parser as an open problem.
