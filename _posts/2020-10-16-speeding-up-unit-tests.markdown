---
layout: post
title:  "Speeding Up Unit Tests"
date:   2020-10-16 01:31:30 +0100
categories: jekyll update
tags: spacy unit-tests parameterized testing
---

I was writing a simple unit test for module that used [Spacy's](https://spacy.io/) NER engine to process text, but noticed the test was taking a really long time to run for each example.

```python
from unittest import TestCase
from parametermized import parameterized
from path.to.text_processing_class import TextProcessor 


def TestMyClass(TestCase):

    def setUp(self):
        self.text_processor = TextProcessor() # loads spacy

    @parameterized([
        {"input": "some text", "output": "processed text"},
        .... #50+ test cases
    ])
    def test_processing(self):
        assert_equal(text_processer(self.input), self.output)

```

Then I realized it was because the `setUp` method is executed before every test. Therefore the `TextProcessor` class was being re-instantiated and therefore re-loading the spacy model before every test case which ends up being very slow. As a simple optimization, I switched to using the `setUpClass` method which only executes once for the class so the same Text Processor instance will be used across test examples. Problem Solved! 
 
```python
#!/usr/bin/env python3

from unittest import TestCase
from parametermized import parameterized
from path.to.text_processing_class import TextProcessor 


def TestMyClass(TestCase):

    @classmethod
    def setUpClass(cls):
        cls.text_processor = TextProcessor() # loads spacy NER

    @parameterized([
        {"input": "some text", "output": "processed text"},
        .... #50+ test cases
    ])
    def test_processing(self):
        assert_equal(text_processer(self.input), self.output)

```
