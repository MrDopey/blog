--- 
date: 2025-01-18T11:46:11Z
draft: true
title: "Hello this is my first blog"
description: "Well yes sir, this is a blog"
slug: "bob bob bob"
authors: []
tags: []
categories: ["code"]
externalLink: ""
series: []
---

{{<tabgroup>}}
{{< tab name="Hello" >}} Hello World! {{< /tab >}}
{{< tab name="Good bye" >}} Goodbye World! {{< /tab >}}

{{</tabgroup>}}


{{< mermaid >}}
flowchart TD
    A[Christmas] -->|Get money| B(Go shopping)
    B --> C{Let me think}
    C -->|One| D[Laptop]
    C -->|Two| E[iPhone]
    C -->|Three| F[fa:fa-car Car]
{{</mermaid>}}


{{< notice warning "Yo hold up" >}}
One note here.
{{< /notice >}}

## Introduction

This is **bold** text, and this is *emphasized* text. :wave:

[emojis](https://gohugo.io/quick-reference/emojis/ 'emojjis')

```ts
const a = [];
a.find(x => x === 'hello')
```


```scala
val x = 15
val z = abc(x)(y)

def bobby(a: Int) String {
    (a + 10).toString
}
```

[the other blog]({{< relref path="posts/2025/01/verifying-docker-caching" >}})
