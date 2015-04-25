---
layout: post
title: "Hooking Variadic Functions With Substrate"
date: 2015-04-24 12:02:06 -0700
post_author: Alban Diquet
categories: ios
---

If you're used to writing iOS tweaks using theos and the [Logos preprocessor directives][logos-wiki], you may run into problems when trying to hook methods or functions that accept an arbitrary number of arguments. One example of a variadic method is _NSString_'s`+ (instancetype)stringWithFormat:(NSString *)format, ...`.

To hook variadic methods or C functions, the Substrate C API has to be used directly. I wrote a quick proof of concept for the function [`open()`][open-man]:

    // Hook open() to catch the path of all files being accessed
    static int (*original_open)(const char *path, int oflag, ...);

    static int replaced_open(const char *path, int oflag, ...) {
        int result = 0;

        // Handle the optional third argument
        if (oflag & O_CREAT) {
            mode_t mode;
            va_list args;

            va_start(args, oflag);
            mode = (mode_t)va_arg(args, int);
            va_end(args);
            result = original_open(path, oflag, mode);
        }
        else {
            result = original_open(path, oflag);
        }
        NSLog(@"OPEN: %s", path);
        return result;
    }

    void hookLibC(void) {
        MSHookFunction(open, replaced_open, (void **) &original_open);
    }


The full code is available [as a gist][poc-gist] on GitHub.

[open-man]: https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man2/open.2.html
[poc-gist]: https://gist.github.com/nabla-c0d3/f952c6fcc1e9d359dbfe
[logos-wiki]: http://iphonedevwiki.net/index.php/Logos
