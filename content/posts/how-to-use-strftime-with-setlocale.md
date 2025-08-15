---
title: "How-to use strftime with setlocale"
date: "2025-08-15T14:53:21+02:00"
tags:
- C
- setlocale
- strftime
- libc
- internationalization
- odin
---

Recently I was puzzled when I was attempting to use `libc.strftime` in Odin,
trying to understand how to make it behave respecting the system localization
setting. You can [see here the forum thread where I posted the question](https://forum.odin-lang.org/t/why-is-strftime-not-respecting-locale/1201).

I was calling `libc.setlocale(.LC_ALL, nil)` expecting it to set the program locale according to the environment variables, and then running `libc.strftime("%B")` expecting it to print the current month according to my system internationalization settings (set to French for that).

It turns out I was misunderstanding the C API of the `libc.setlocale` function,
which has different behavior if you give it a `NULL` value (`nil` in Odin)
or an empty string (`""`). It's all there [explained in its manual page](https://www.man7.org/linux/man-pages/man3/setlocale.3.html).

* If you give it `NULL`, `setlocale()` will merely return the current locale, NOT modifying the current locale.

* If you give it an empty string `""`, `setlocale()` will set the current locale respecting the system environment localization settings via the `LC_*` variables.

Once I changed my program to pass the empty-string like so `libc.setlocale(.LC_ALL, "")`, then `libc.strftime("%B")` behaved as expected.

In Python, we have a nice API through the [`locale`](https://docs.python.org/3/library/locale.html) and [`datetime`](https://docs.python.org/3/library/datetime.html) modules that wrap around this stuff, I guess I had lost touch with the C way.

Here's a C program that exemplifies how this work:

```
#include <locale.h>
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

void print_locale_and_date(const char *locale_string, const char *date_format) {
  char outstr[200];
  time_t t;
  struct tm *tmp;

  t = time(NULL);
  tmp = localtime(&t);
  if (tmp == NULL) {
    perror("localtime");
    exit(EXIT_FAILURE);
  }

  char *cur_locale = setlocale(LC_ALL, locale_string);
  printf("Current locale is \"%s\"\n", cur_locale);

  if (strftime(outstr, sizeof(outstr), date_format, tmp) == 0) {
    fprintf(stderr, "strftime returned 0");
    exit(EXIT_FAILURE);
  }

  printf("strftime result was: \"%s\"\n\n", outstr);
}

int main(int argc, char *argv[]) {
  const char *date_format = "%d %B, %Y";

  // If given NULL, setlocale will just return the current locale
  print_locale_and_date(NULL, date_format);

  // But if given empty string, it will set the locale to the default locale
  // according to the environment LC_* variables (in my system, it is
  // configured to use French for LC_TIME)
  print_locale_and_date("", date_format);

  // US English locale
  print_locale_and_date("en_US.UTF-8", date_format);

  // If given NULL, setlocale will just return the current locale
  print_locale_and_date(NULL, date_format);

  // French locale
  print_locale_and_date("fr_FR.UTF-8", date_format);

  // Back to default locale
  print_locale_and_date("", date_format);

  exit(EXIT_SUCCESS);
}
```

Note that it has some locales hardcoded, if you're not getting the output in French where expected language, run: `sudo locale-gen fr_FR.UTF-8` to generate the system localization files for it.
