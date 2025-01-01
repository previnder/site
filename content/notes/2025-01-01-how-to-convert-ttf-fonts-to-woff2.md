---
title: How to convert TTF fonts to WOFF2
date: 2025-01-01
slug: /ttf-to-woff2
---

When you download a font from a source like [Google Fonts](https://fonts.google.com/), it’s likely in either [TTF](https://en.wikipedia.org/wiki/TrueType) or [OTF](https://en.wikipedia.org/wiki/OpenType) formats, the two most widely used font formats in the world. If you’re going to use these fonts on the web, however, it’s a good idea to convert them to [WOFF2](https://en.wikipedia.org/wiki/Web_Open_Font_Format) first, because WOFF2 fonts are significantly smaller in size (WOFF2 fonts are the same as TTF or OTF fonts but compressed).

Of course, in many cases (as with Google Fonts) one can just use a CDN, but it’s generally a good idea to go to the trouble of self-hosting fonts (on the same domain as your site), rather than using a CDN, as then the fonts would be delivered to the end-user slightly faster when they visit your site. That’s because accessing a CDN involves making additional network requests, DNS lookups, fetching additional CSS files, etc.

Fortunately for us, Google has an easy-to-use [open-source program](https://github.com/google/woff2) for converting between TTF/OTF and WOFF2 formats. We can just clone the repository, build the program, and use it:

```shell
# Clone and build the program
git clone --recursive https://github.com/google/woff2
cd woff2
make clean all

# Use it to convert a TTF/OTF font to WOFF2
./woff2_compress ~/Download/font.ttf
```

To install the program on your computer (assuming of course that it’s Unix-based), just copy the `woff2_compress` and `woff2_decompress` binaries to a directory in your `$PATH` (on Linux, I would just copy them to `/usr/bin`).
