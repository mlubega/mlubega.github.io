---
layout: post
title:  "Level Up: Scripting in Python with Click"
date:   2020-10-12 23:55:30 +0100
categories: jekyll update
---
You know that feeling when you ask someone to beep you on your pager, but they give you a funny a look and ask if you know what a phone is? Well, this hasn’t happened to me, but it felt much the same when I learned we could now use [Click](https://click.palletsprojects.com/en/7.x/) to build python scripts.

In the past I might’ve used written a script using [argparse](https://docs.python.org/3/howto/argparse.html) like so:

```python
#!/usr/bin/env python3

import argparse


def main(num, input_file, output_dir):
    # processing code


if __name__ ==  "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("num", help="number of things to count")
    parser.add_argument("-s", "--src_file", required=True,  help="path to source file", type=str)
    parser.add_argument("-d", "--dst_dir", required=True,  help="path to destination folder", type=str)
    args = parser.parse_args()

    main(args.num, args.src_file, args.dst_dir)
```

Now I can use (in my opinion) the much lighter, more elegant Click interface to do the exactly same thing:


```python
#!/usr/bin/env python3

import click


@click.command()
@click.argument("num", type=click.INT, help="number of things to count")
@click.option("-s", "--src_file", required=True, type=click.File(mode="r"), help="path to source file")
@click.option("-d", "--dst_dir", required=True, type=click.Path(exists=True, dir_okay=True), help="path to destination folder")
def main(num: int, src_file: str,  dst_dir: str):
    # processing code


if __name__ == "__main__":
    main()
```

The `@click.command` decorator makes a function into a cli command, `@click.argument` is for positional params, `@click.option` is for optional/keyword params, and you can enable additional validation of inputs via types like `click.File()` and `click.Path()` to check if a file or directory exists, is readable/writeable, and much more. Finally, one additional and perhaps overlooked benefit of Click is its super simple api for testing:

```python
from unittest import TestCase
from click.testing import CliRunner
from path.to.my.script import main


class TestScripts(TestCase):

    def test_my_script(self):
        runner = CliRunner()
        output = runner.invoke(main, [ 2, "-s", "input_dir/somefile.txt", "-d", "output_dir"])
        assert output.return_code == 0

```
Argparse doesn’t have an equivalent functionality and therefore testing scripts that rely on argparse can quickly become complicated.

Head on over the [Click Documentation](https://click.palletsprojects.com/en/7.x/) to read up on all the additional cool features I didn’t tell you about!
