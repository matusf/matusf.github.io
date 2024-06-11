+++
title = "Squeezing the most out of argparse"
date = 2021-07-08
authors = ["Matúš Ferech"]
+++

In this post, I would like to argue that Python's `argparse` is often the right tool for the job, and you do not need to install additional CLI argument parsers. The straightforward reason to choose it might be that you want to write a simple script that you pass to your colleagues, and you do not want to bother them with the installation of dependencies. You want to make it as portable as possible. However, I will try to show you that there are other ones.

## Getting variables from env as well

Loading configuration from the environment is one of the [prefered ways](https://12factor.net/config) to configure applications. With `argparse`, you can load variables from both the environment as well as from the command line. Good use for this combination is when you need to load secret variables. Secret variables should be loaded from the environment since anyone can inspect running processes (`ps`) that include all CLI arguments and thus see the secrets.

```py
from os import getenv
from argparse import ArgumentParser

p = ArgumentParser()
p.add_argument('--port', default=getenv('PORT'))
```

### Making sure variables are loaded

Now that we can load a variable from the environment we want to make sure that the variable is passed either as a command-line argument or as an environment variable. Adding `required=True` to `add_argument` does not help since then the default option is ignored. However, there is a neat trick we can do. The command-line argument will be required if we have not found the variable in env.

```py
from os import getenv
from argparse import ArgumentParser

p = ArgumentParser()
p.add_argument('--port', default=getenv('PORT'), required=not getenv('PORT'))
```

## Typed environment variables

The main advantage of `argparse` for me is that you can have typed environment variables. No need to convert environment variables to the desired type and manually handling exceptions. The type can be specified with a `type` keyword argument. Actually, `type` can be any callable that takes a string and returns the desired type. If it raises `TypeError` or `ValueError` a nice error message is displayed.

You can take it one step further by writing your own parse function. If the parsing fails, raise an `ArgumentTypeError` with a help message which will be shown to the user. The following example shows how to parse a variable from the environment with additional constraints using `argparse`.

```py
from os import getenv
from argparse import ArgumentParser, ArgumentTypeError


def parse_port(n):
    port = int(n)
    if port < 0:
        raise ArgumentTypeError('must be non-negative')
    return port


p = ArgumentParser()
p.add_argument(
    '--port',
    default=getenv('PORT'),
    required=not getenv('PORT'),
    type=parse_port,
)
p.parse_args()
```

After running it, we see a nice help message.

```txt
python x.py --port -1
usage: x.py [-h] --port PORT
x.py: error: argument --port: must be non-negative
```

---

There is one gotcha though. Loading boolean variables from the environment and specifying `type` as `bool` is not sufficient since every non-empty string is considered to be true (even `"false"`, `"no"` etc.). Therefore we need to use a different function, like `strtobool` as shown below.

```py
from os import getenv
from argparse import ArgumentParser
from distutils.util import strtobool

p = ArgumentParser()
p.add_argument('--foo', default=getenv('foo'), type=lambda x: bool(strtobool(x)))
```

The result of `strtobool` is wrapped in `bool` because unfortunately, it returns an int instead of bool (for [historical reasons](https://bugs.python.org/issue27721)).
