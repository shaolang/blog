---
title: "pip Behind Corporate Firewalls"
date: 2020-01-28T12:17:59+08:00
allowComments: true
---

Corporate firewalls... never mind, that's a rant for another day.

To appease the corporate firewalls for pip, create `%userprofile%\pip\pip.ini`
with the following content:

```ini
[global]
trusted-host = pypi.python.org
               pypi.org
               files.pythonhosted.org
```

Set the `https_proxy` and `http_proxy` environment settings, and `pip install`
away to your heart's content.

Note that this solution works since pip 10.0, and yup, works at least within
the corporate firewalls that I'm behind currently.
