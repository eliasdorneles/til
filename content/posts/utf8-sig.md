---
title: "How to avoid BOM issues with CSV files created on Windows"
draft: false
date: 2023-10-27T15:00:00.000Z
description: ""
tags:
- python
- csv
- microsoft
---

When loading CSV files in Python that were created on Windows, you can get some
weird errors because the Microsoft tools usually add a
[BOM](https://en.wikipedia.org/wiki/Byte_order_mark).

You can workaround that by using the encoding `'utf-8-sig'`, which will work
also for regular UTF-8 files without the mark. Like this:

```python
with open('MY_FILE.csv', encoding='utf-8-sig'):
    data = list(csv.DictReader(f))
```
