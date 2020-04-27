---
title: "Python Code Structure and Distribution"
date: 2018-07-13T23:30:15+08:00
allowComments: true
tags: [python]
---

Probably it's due to my inexperience in Python's community/ecosystem, I find
it amusing that Python community has no agreed standard in code organization.
Yeah, [The Hitchhiker's Guide to Python!][hitchhiker] attempts to recommend
some [structure][hitchhiker-structure], but the popular [pytest][pytest]
recommends [something entirely different][pytest-structure].

By adopting pytest's recommendation--i.e., putting code in `src` directory--I
need to make the some changes, changes that are not obvious to a noob like me:

* As my code modules are no longer on `PYTHONPATH`, I need to install
  [pytest-pythonpath][pytest-pythonpath] and add `src` to in `pytest.ini` to
  inject my code into `PYTHONPATH` for pytest to function.
* Similarly, I need to update my cx_Freeze `setup.py` to point to the entry
  file in `src` (highlighted):

{{< highlight python "linenos=inline,hl_lines=21">}}
# Assuming the filename is setup.py
import os.path
import sys
from cx_Freeze import setup, Executable

PYTHON_INSTALL_DIR = os.path.dirname(os.path.dirname(os.__file__))
os.environ['TCL_LIBRARY'] = os.path.join(PYTHON_INSTALL_DIR, 'tcl', 'tcl8.6')
os.environ['TK_LIBRARY'] = os.path.join(PYTHON_INSTALL_DIR, 'tcl', 'tk8.6')

base = 'Win32GUI'

options={'build_exe': {
   'includes': ['tkinter'],
   'include_files': [
       './resources/tcl86t.dll',
       './resources/tk86t.dll',
       './resources/vcruntime140.dll'],
    'path': sys.path + [os.path.abspath('./src')]}}

setup(  name = 'my_awesome_app',
       version = '0.1.0',
       description = 'My Super, Duper Awesome App',
       executables = [Executable(script='./src/main.py', base=base,
           targetName='awesome.exe',
           copyright='Copyright © 2018 Shaolang Ai')],
       options = options)
{{< /highlight >}}

_Yes, I develop apps to run in Windows._ The code above also shows the things
I need to include to ensure that the packaged app can run TkInter without
issues. With these in place, running `python setup.py build` will
create my redistributable standalone app correctly.


[hitchhiker]: http://docs.python-guide.org/en/latest/
[hitchhiker-structure]: http://docs.python-guide.org/en/latest/writing/structure/
[pytest]: https://pytest.org
[pytest-structure]: https://docs.pytest.org/en/latest/goodpractices.html#tests-outside-application-code
[pytest-pythonpath]: https://github.com/bigsassy/pytest-pythonpath
