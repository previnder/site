---
title: Make KDE Plasma’s file indexer, Baloo, usable
date: 2025-04-28
slug: /make-baloo-usable
---

KDE Plasma is great, but the file indexer that ships with it, Baloo, is not. In fact, it’s terrible. If I leave it as it is, one of its processes, for some reason, inevitably starts using 100% CPU, which I only notice once my laptop starts heating up. You’d think this only happens for a short period of time when a lot of new files are waiting to be indexed at once, but that doesn’t seem to be the case, from what I can tell.

I’ve found that there’s a way to _kind-of_ fix this, by making only the most minimal use of the file indexer. But each time I install KDE Plasma, in these many years I’ve been using it, I always forget to configure that, which is why I’m writing this note.

The way to fix Baloo is to add the following lines to the `baloofilerc` file[^1] (located in `$HOME/.config`).

```conf
[General]
only basic indexing=true
```

This instructs Baloo to only index filenames, which is good enough for me. This way, Baloo indexes all the files it’s got to index extremely quickly, and the high CPU usage problem goes away completely.

After adding those lines to the `baloofilerc` file, run `balooctl6 disable` and then `balooctl6 purge`, which will first disable the file indexer and then delete its database. If any Baloo processes are still running after that (run `ps aux | grep baloo` to check), kill them manually. Then restart Baloo by running `balooctl6 enable`.

Run `balooctl6 status` to check the status of the file indexer. Baloo should trouble you no more now.

[^1]: See the [Baloo documentation page](https://community.kde.org/Baloo/Configuration) for more infomation.
