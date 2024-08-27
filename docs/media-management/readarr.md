---
title: Readarr
permalink: /docs/media-management/readarr/
---

Since Bazarr was a breeze I'm feeling good for Readarr. Only hang up I can think of will be around needing two of these, but each can use the same port since I'll load balance them so the config should be identical. 

## CRDs

Namespace will be `media-management` and we will use the `app-template` chart. For this arr we need two, otherwise we can't select between audio and ebook. For some reason they designed this as if an audio book was a 4K version of an ebook.

`helmrelease-readarr-ebooks.yaml`
```yaml
```

`helmrelease-readarr-audiobooks.yaml`
```yaml
```