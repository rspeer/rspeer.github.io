---
layout: post
title: "U+DEADBEEF: Why you shouldn't trust arbitrary text encodings"
date: 2014-03-30 15:22:50 -0400
comments: true
categories: [unicode, python, glitches]
---
A few months ago I posted a [Gist about ways to totally break Python's Unicode representation](https://gist.github.com/rspeer/7559750), by exploiting [bug #19279](http://bugs.python.org/issue19279) in its UTF-7 decoder.

You might not have heard much about UTF-7. It doesn't have very much at all to do with [UTF-8](http://en.wikipedia.org/wiki/UTF-8), the well-designed (by Ken Thompson) representation of Unicode that's taking over the world. [UTF-7](http://en.wikipedia.org/wiki/UTF-7) is a poorly-designed, obsolete proposal for encapsulating Unicode inside ASCII itself.

The Python bug (which is fixed now) is that the encoder doesn't reset its state correctly when it encounters an erroneous UTF-7 sequence. You can make these errors pile up, until they add up to an impossible character, at codepoint U+DEADBEEF.

(While Unicode codepoints are often represented as 32-bit integers, not all 32-bit integers are Unicode codepoints. The highest possible Unicode codepoint is U+10FFFF.)

Once you have this impossible character in a string, you can pass it to standard library functions to cause all sorts of data corruption, including making a SQLite database unreadable.

There's one easy lesson to take from this: keep Python up to date. If you're on Python 2.7.6 or 3.3 or later, you won't encounter this particular bug. But then, this isn't [the last UTF-7 bug](http://bugs.python.org/issue20538) in Python.

Here's the more important lesson, though, which applies in any programming language:

## Don't let someone else's data tell your code what to do

When people initially uncover these encoding bugs, it is often because they are [scraping the Web](http://bugs.python.org/issue19279#msg200117), or e-mails, or some other format where you can tell the recipient what encoding your text is in. There's lots of code out there that will read that metadata and use the appropriate decoder in Python.

This is kind of dangerous. You're letting data you didn't create determine what your code does. If a Web page says it's in UTF-7, it'll use the UTF-7 decoder, which you've probably never used before. If it says it's in `gzip_codec` and contains a [gzip bomb](https://mail.python.org/pipermail/python-dev/2013-November/130188.html), boom.

Some of these attacks are theoretical, but there are a few real web pages out there that -- perhaps accidentally -- claim to be in UTF-7 and trigger bug #19279. Really, it shouldn't matter what the Web page you're scraping claims: your code has *no business* firing up the UTF-7 decoder, which it usually doesn't do, just because some Web page told it to.

## UTF-7 is a vulnerability

<!-- more -->

Even if the decoder were bug-free, UTF-7 would be a totally unsuitable encoding for exchanging data. The fact that you can hide ASCII characters as other strings in UTF-7 has been used in [a real XSS vulnerability](http://nedbatchelder.com/blog/200704/xss_with_utf7.html) that was only fixed by major Web browsers disabling automatic UTF-7 decoding.

```
+ADw-script+AD4-alert(+ACc-xss+ACc-)+ADw-+AC8-script+AD4-
```

UTF-7 isn't even necessary to solve the problem it was supposed to solve (data transfer over an ASCII-only protocol). JSON can encode any Unicode character in plain ASCII, if it needs to, with its escape syntax:

```python
>>> string = "Here's a snowman: \N{SNOWMAN}"
>>> print(string)
Here's a snowman: ☃

>>> import json
>>> print(json.dumps(string))
"Here's a snowman: \u2603"
```

## Make a whitelist of encodings

Is the solution to blacklist UTF-7, `gzip_codec`, `bzip2_codec`, and so on? It would take a lot of research to find out the complete list of encodings that you're better off avoiding. Just at a glance, I see that Python supports HZ, a Chinese encoding with UTF-7-like properties. Do we know what its quirks and vulnerabilities are? In what circumstances would you want to find out?

When you are scraping the Web, accepting just any encoding is absurd. UTF-8 has almost entirely taken over the Internet. There are a few other encodings you'll have to deal with from time to time, but you probably don't have to deal with every kind of ugly data in every language. So perhaps you should *whitelist* non-UTF-8 encodings when you know you have to decode them.

Here are the encodings that I personally think are used often enough to be safe:

- `ASCII` (obviously)
- `UTF-8` (the ASCII-compatible encoding preferred by nearly everything on the Web)
- `UTF-16` (preferred by Windows APIs)
- `ISO-8859-1` as a special case (pretend they said `Windows-1252`, because that's almost always what people mean when they say "ISO-8859-1")
- `Windows-1250` through `Windows-1258`, `ISO-8859-1` through `ISO-8859-15`, and `MacRoman` (although some of these are terribly obscure, they all use the same code)
- `cp437` (the old IBM character set that's still, to this day, baked into every PC's display hardware)

This covers all the easy cases, leaving us with the mess of encodings that China, Japan, and Korea have created and continue to use because they're too proud to adopt UTF-8. If you intend to have solid support for a CJK language, where you'll probably encounter other problems besides just decoding the text, add some more encodings to your whitelist:

- If you support Japanese text, you should support `Shift-JIS` and `EUC-JP`.
- If you support Korean text, you should support `EUC-KR`.
- If you support Traditional Chinese text, you should support `Big5`.
- If you support Simplified Chinese text, you should support `GBK`.

I don't know enough about `GB18030`, the Chinese-centric alternative to UTF-8. It seems incredibly complex, and although it's the official standard of the People's Republic of China, I get the idea that it's not used in practice that much.

## The U+DEADBEEF code

{% include_code deadbeef_character.py %}
