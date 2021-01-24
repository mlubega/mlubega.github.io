---
layout: post
title:  "Fiddling around with Python Generators"
date:   2020-11-03 02:29:30 +0100
categories: python generators coroutines html lxml
---
I was recently working with a bit code that traversed the HTML tree and did some processing on the text contained within:

```python
def process_html(html):
    tree = etree.parse(StringIO(html_content), etree.HTMLParser)
    for element in tree.iter():
        if element.text:
            element.text = process_text(element.text)
        if element.tail:
            element.tail = process_text(element.tail)
        for attrib in element.attrib:
            if element.get(attrib):
                element.set(attrib, process_text(element.get(attrib))
```
It’s functional, but I wondered if there was a way to reduce the number of times `process_text` is called since I essentially want to do the same text processing on all text elements. Well, I had recently been reading up on python generators and decided to do a little experiment to see if they could be of use here. 

Generators are iterable objects that yield data items one a time, which comes in handy when working with data that are too large to fit into memory. Instead of loading the entire list into memory, you can instead process items one at a time.

```python
regular_list = [ x for x in my_list ] # all in memory
generator_list = ( x for x in my_list ) # only loads and yields one x at a time
```
It’s also possible to define a generator as a function:
```python
def generate_infinite_nums():
    x = 0
    while true:
        yield x
        x += 1

# bad idea - infinite sequence
for num in generate_infinite_nums:
    print(num)

# better idea - advance the iterator once
num = next(generate_infinite_nums):
```

I wondered if I could refactor the html traversal into a generator function that yields text elements and therefore only have to call the `process_text` function only once instead of three times. So I tried something like this:

```python
def traverse_html(tree):
    for element in tree.iter():
        if element.text:
            element.text =  yield element.text 
        if element.tail:
            element.tail = yield element.tail 
        for attrib in element.attrib:
            if element.get(attrib):
                element.set(attrib,  (yield element.get(attrib)) )

def process_html(html):
    tree = etree.parse(StringIO(html_content), etree.HTMLParser)

    html_traverser = traverse_html(tree)
    for extracted_text in html:
        html_traverser.send(process_text(extracted_text))
```
Here, I’ve used the `yield` function which yields the text to the caller before resuming execution. The `.send()` allows us to pass processed text back into the generator function, enabling us to update the element at line `element.text = yield element.text`. When I ran this code, however, every second element was unexpectedly `None`. But why?

It turns out when using a for loop to iterate over the generator, it’s calling `next()` on the generator which is equivalent to `.send(None)` (see this [Stackflow thread](https://stackoverflow.com/a/12638313)). So I’m inadvertently sending data twice -- `.send(None)` when the `for` statement executes and `.send(process_text(extracted_text))` within the for loop -- which explains why every other value was `None`. 

One possible [solution](https://stackoverflow.com/questions/36835782/using-generator-send-within-a-for-loop)  is re-writing the function to iterate over the `.send()` without the for loop like so:
```python
def process_html(html):
    tree = etree.parse(StringIO(html_content), etree.HTMLParser)

    html_traverser = traverse_html(tree)
    try:
        text = next(html_traverser)
        while true:
            text = html_traverser.send(process_text(text))
    except StopIteration:
        pass
```
Since we’re now using a while loop, we need to explicitly catch the `StopIteration` exception that generators raise after the data stream is exhausted (the for loop handled this automatically for us).

In the end, I didn’t incorporate this code change. The function was ultimately re-written to support batch processing. Still, it was a fun exercise in understanding generators!

 


&nbsp;
#### Further Reading
---
[How to Use Generators and yield in Python](https://realpython.com/introduction-to-python-generators/)
