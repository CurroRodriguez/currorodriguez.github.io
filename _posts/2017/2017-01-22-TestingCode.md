---
layout: post
title: Testing Code
category: added
tags:
    - foo
    - bar
    - zap
---

Here I will test the different options to put code in an article. I'll
start by just using markdown:

```
def do_something(a, b):
    return a + b
```

This example specifies the language of the code:

```python
def do_something(a, b):
    return a + b
```

Here is some C# code:

```csharp
public class MyClass
{
    public string Name {get;set;}
    public int Add(int first, int second)
    {
        return first + second;
    }
}
```

Here I try to embed a gist:

<script src="https://gist.github.com/CurroRodriguez/c81beb6e8e7841e73d38.js"></script>
