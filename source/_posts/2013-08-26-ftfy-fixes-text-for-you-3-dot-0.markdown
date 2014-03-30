---
layout: post
title: "ftfy (fixes text for you) version 3.0"
date: 2013-08-26 18:27:44 -0400
comments: true
categories: [python, unicode, software, luminoso]
---
About a year ago, Luminoso blogged about how to ungarble garbled Unicode in a post called [Fixing common Unicode mistakes with Python â€” after they’ve been made](http://blog.luminoso.com/2012/08/20/fix-unicode-mistakes-with-python/). Shortly after that, we released the code in a Python package called [ftfy](https://github.com/LuminosoInsight/python-ftfy).

You have almost certainly seen the kind of problem ftfy fixes. Here’s [a shoutout](http://isabelcastillo.com/international-characters-encoding-fpdf) from a developer who found that her database was full of place names such as “BucureÅŸti, Romania” because of someone else’s bug. That’s easy enough to fix:

```python
>>> from ftfy import fix_text
 
>>> print(fix_text(u'BucureÅŸti, Romania'))
Bucureşti, Romania
  
>>> fix_text(u'Sokalâ€™, Lâ€™vivsâ€™ka Oblastâ€™, Ukraine')
"Sokal', L'vivs'ka Oblast', Ukraine"
```

[Read more on the Luminoso blog](http://blog.luminoso.com/2013/08/26/ftfy-fixes-text-for-you-version-3-0/)
