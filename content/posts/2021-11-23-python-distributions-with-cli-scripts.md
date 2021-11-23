---
title: "Python Distributions With CLI Scripts"
date: 2021-11-23T08:50:30+08:00
allowComments: true
tags: [python]
---

Let's create a "hello, world" distributable that creates a CLI executable,
`greet`: first, some preparation:

```bash
$ mkdir hello-world
$ cd hello-world
$ python -m venv hello-world      # create a virtual environment, hello-world
$ hello-world/Scripts/activate    # activate the virtual environment
$ mkdir -p src/hello/scripts
$ touch src/hello/__init__.py
```

Create the `greet` function in `src/hello/__init__.py`:

```python {linenos=table}
def greet(name):
    print(f'Hello, {name}')
```

Nothing earth-shattering for our purpose here. Next, create the CLI script
that calls `greet` when an argument is given on the command line:

```python {linenos=table}
# in src/hello/scripts/cli.py

from hello import greet
import sys

def main():
    if len(sys.argv) > 1:
        greet(sys.argv[1])
    else:
        print(f'Please specify a name')
```

Note the function name `main`: it's an arbitrary name but one that
`setup.cfg`[^1] references so [`build`][build] can generate the
executable. Next, let's create a minimal `setup.cfg`.

```plaintext {linenos=table, hl_lines=["4-6","11-13"]}
[metadata]
name = hello-world

[options]
package_dir = =src
packages = find:

[options.packages.find]
where = src

[options.entry_points]
console_scripts =
  greet = hello.scripts.cli:main
```

A few notable points:

* Lines 4-6 are necessary when [using a `src/` layout][src-layout]
* Lines 11-13 specifies the CLI command `build` should generate in the
  virtual environment's `Scripts` directory
  * Line 13 takes the form of `executable-name = package.module:function`;
    in this case, the executable's name is `greet` and it invokes
    the function `main` in module `hello.scripts.cli`.

Next, [create `pyproject.toml`][create-pyproject.toml] that `build`
relies on to build the distributable:

```toml {linenos=table}
[build-system]
requires = ["setuptools >= 42", "wheel"]
build-backend = "setuptools.build_meta"
```

Finally, it's time to build the distributable:

```bash
$ pip install build       # installs the build tool
$ python -m build --wheel # builds only the wheel distribution
```

All's done. To test this, create another virtual environment, install
the distributable found in `dist/` with `pip`; this generates the executable
`greet` in the environment's `Scripts/` directory.

[^1]: According to [Configuring metadata][config-metadata], `setup.cfg` is
      the preferred format.

[build]: https://github.com/pypa/build
[src-layout]:https://setuptools.pypa.io/en/latest/userguide/declarative_config.html#using-a-src-layout
[create-pyproject.toml]: https://packaging.python.org/tutorials/packaging-projects/#creating-pyproject-toml
[config-metadata]: https://packaging.python.org/tutorials/packaging-projects/#configuring-metadata
