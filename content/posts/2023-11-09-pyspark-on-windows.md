---
title: "PySpark on Windows"
date: 2023-11-09T11:22:33+08:00
draft: false
allowComments: true
---

Setting PySpark up on Windows[^1] for local runs is relatively easy...
when you know the supplementary steps and hidden requirements necessary
to make it work.

As one of the dependencies is [winutils][winutils][^2], the Spark version that
can run on Windows depends on the maintainer's release of `winutils.exe`.
For example, as of 10 Nov 2023, because the latest winutils version is 3.3.5,
Windows users can only use Spark 3.3.x, i.e., Spark's MAJOR.MINOR version
must match winutils' MAJOOR.MINOR version.

'Nuff said, the steps to setting up PySpark on Windows are as follows:

1. Download winutils.exe from winutils repo.
2. Download Spark that matches winutils' (MAJOR.MINOR) version.
3. Unzip Spark and copy winutils to Spark's `bin` directory.
3. Set the environment variables `SPARK_HOME` and `HADOOP_HOME` to that Spark
   directory.
4. Download and unzip [Java Development Kit][jdk] (JDK).
5. Set the environment variable `JAVA_HOME` to that JDK directory.
6. Add `%SPARK_HOME%\bin%` and %JAVA_HOME%\bin` to `PATH` environment variable.
7. Create a virtualenv/venv[^3] and pip install pyspark (again, the
   (MAJOR.MINOR)
   version must match winutils').
8. Navigate to the Scripts directory of the newly created virtualenv/venv
   and copy `python.exe` as `python3.exe`.

PySpark should work now; or does it? Nope, two caveats as of Nov 2023:

1. The latest Python version PySpark supports is 3.10, i.e., 3.11 and 3.12
will error out for certain operations, e.g., `.createDataFrame()`.
2. The latest JDK Spark supports is 17.0, i.e.g, newer versions like the
latest Long Term Support (LTS) JDK may cause issues.


[^1]: Yes, Windows. Don't judge. Most companies in Asia still issue Windows laptops.
[^2]: Another way is to use [Hadoop Bare Naked Local FS][hbnlfs] with [some (non-trivial) code][issue2] to get it running.
[^3]: You develop with virtualenv/venv, don't you? Do you?!

[winutils]: https://github.com/cdarlint/winutils
[hbnlfs]: https://github.com/globalmentor/hadoop-bare-naked-local-fs
[issue2]: https://github.com/globalmentor/hadoop-bare-naked-local-fs/issues/2#issuecomment-1443768384
[jdk]: https://openjdk.org
