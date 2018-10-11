# Patching Source Code to Conda Build Recipes

If you are a developer who relies upon conda, I hope to encourage you to begin building your own packages so that your projects can be used just like all of the other packages you rely upon. The success of Anaconda rests upon the ease to search for, install, and create environments for packages while automatically managing dependencies and versions. Conda Recipes are the blueprint to building packages. They contain all the dependency management information, and they are typically very easy to create. 

[tip]: https://openclipart.org/image/24px/svg_to_png/194429/cartoon-eyes.png

Of course, sometimes things need a little help. If a recipe fails to build, there are many ways to fix it. Usually, you have permission to change both the project's recipe, and its source code. In this post, we focus on how to create a working conda recipe when you cannot _directly_ change the source code. Say, for example, that you wish to create a recipe for a tagged version in which you've found a bug.  Along the way, we describe a few common cookbook tips to ensure your recipe works. Look for this symbol ![cookbook tip][tip]

When one cannot, or wishes not to, change the code under source control, conda-build provides the ability to specify a patch file. A [patch](https://en.wikipedia.org/wiki/Patch_(Unix))  consists of a list of changes to apply to a set of text files. These changes are applied during the build, after the source is downloaded, but before the build script runs.The built package contains your change, while the source repo (i.e. in github or pypi) remains unchanged.

Patches are particularly useful in the following scenarios.
* There is a bug in the source code.
* The build is for a new target (e.g. Centos6, or Windows with Visual Studio 2013)
* The build is for a different version of Python than the author intended

In what follows, we present a tutorial that we hope will help you with your case. Some knowledge of conda-build is assumed. For reference, the docs are [here](https://conda.io/docs/user-guide/tasks/build-packages/recipe.html)

You will need conda, conda-build, and git. conda can be downloaded [here](https://conda.io/miniconda.html). Once conda is set up,  use it to install build and git. `conda install conda-build git`. The instructions that follow are written assuming a POSIX system. If you are using Windows, consider downloading MSYS2 to follow along.

### 1. Our starting point - a skeleton recipe

We need a starting point for our recipe. There are three typical choices.
* Manually create a [meta.yaml](https://conda.io/docs/user-guide/tasks/build-packages/define-metadata.html) from scratch or copy an existing one.
* ![conda-forge][tip] Clone an existing recipe from [conda-forge](https://github.com/conda-forge) 
* ![conda-skeleton][tip] Use [conda-skeleton](https://conda.io/docs/user-guide/tutorials/build-pkgs-skeleton.html) to create a recipe from a PyPI package

We happened to stumble upon a package in PyPI that required patching to create a recipe for. The internals of the package aren't important. As the package is on PyPI, we will use a skeleton for our starting point.  Let's make a new directory to work within called 'patched',  build our skeleton recipe, and then rename it with mv. 

```
mkdir patched
cd patched
conda skeleton pypi pennies
mv pennies recipe
```

To track the changes we make to our recipe, let's initialize a git repository for it.
```
git init recipe 
cd recipe
git add meta.yaml 
git commit -m "Initial skeleton recipe"
cd ..
```

Finally, let's try to build our auto-generated recipe.

```
conda build recipe
```

### 2. Update recipe 

The build log should throw an exception like the following.

```
Unable to parse meta.yaml file

Error Message:
--> mapping values are not allowed in this context
-->   in "<unicode string>", line 46, column 19
```

This is just a yaml parsing issue. A quick examination of the line it refers to suggests that it doesn't like the second colon in the summary.

Make the following changes to `meta.yaml`

```
-  summary: pennies: pythonic quantitative finance library
+  summary: pythonic quantitative finance library
```

Commit this change
```
git add meta.yaml
git commit -m "Updated skeleton to parse yaml"
cd ..
```

![render][tip] Use conda-render to test that your recipe is consistent. It provides a load of useful information such as the recipe's full url, along with the entire list of required packages and their versions. This will help us make a local copy of the source code. `conda render recipe`

### 3. Create a new git repo for the source. Point recipe to it.

When we rebuild, `conda build recipe`, we get a new exception. 
```
Traceback (most recent call last):
  File "/Users/cclements/miniconda3/conda-bld/pennies_1539210292243/test_tmp/run_test.py", line 5, in <module>
    import pennies.calculators
  File "/Users/cclements/miniconda3/conda-bld/pennies_1539210292243/_test_env_placehold_placehold_placehold_placehold_placehold_placehold_placehold_placehold_placehold_placehold_placehold_placehold_placehold_placehold_placehold_placehold_placehold_placehold_place/lib/python3.6/site-packages/pennies/calculators/__init__.py", line 2, in <module>
    import assets
ModuleNotFoundError: No module named 'assets'
```

This is an issue with the source, not the recipe. It is at this point that we begin patching.

When playing with the source code, it is best to create a local git repo, and make frequent commits while experimenting with building the recipe. Most likely, you will end up making changes in both the source code and the recipe.

When conda-build runs, it creates a local copy of the source code. This can be found in the conda-build directory. The stack trace above shows us where conda-build is working. (Users/cclements/miniconda3/conda-bld/pennies_1539210292243). The source is in a /work directory under there. We can create our local source repo here.

One can also create a local copy from the remote source as given in conda-render output.  In either case, the result is the same. We show the latter, as it is easier to describe.

```
wget https://pypi.io/packages/source/p/pennies/pennies-0.2.0.tar.gz
tar -xvf pennies-0.2.0.tar.gz
mv pennies-0.2.0 src
git init src
cd src
git add .
git commit -m "Original source"
cd ..
```

Update recipe to point to local source. Make the following changes to meta.yaml
```
source:
-  url: https://pypi.io/packages/source/{{ name[0] }}/{{ name }}/{{ name }}-{{ version }}.tar.gz
-  sha256: 5044bfd6009dc4d6d14ed8d8355d0cc4b9cb4aafad105b80b0c481f8beacdc15
+  path: ../src
```

Commit your change.

```
cd recipe
git add meta.yaml
git commit -m "Update source to point to local path"
cd ..
```

Confirm that the same exception is thrown by `conda build recipe`.  

### Fix the bug in the source.

Recall that the exception that we had was a missing import.
```
pennies/calculators/__init__.py", line 2, in <module>
    import assets
ModuleNotFoundError: No module named 'assets'
```

No problem. Fix up the complaining file, and rerun the build. 
```
$ git diff
diff --git a/pennies/calculators/__init__.py b/pennies/calculators/__init__.py
index 482d915..46f4a41 100644
--- a/pennies/calculators/__init__.py
+++ b/pennies/calculators/__init__.py
@@ -1,5 +1,6 @@
 # import modules that define dispatched methods, eg presentvalue, par_rate
-import assets
-import payments
-import swaps
-import trades
\ No newline at end of file
+from . import assets
+from . import payments
+from . import swaps
+from . import trades
```

### Create a patch

Having confirmed that our change to the source code has fixed the build, we commit the change, and then use git to produce a patch file.
```
cd src
git add pennies/calculators/__init__.py
git commit -m "fix imports"
git format-patch -1
````

This produces a file named '0001-fix-imports.patch'. It shows the difference between the latest and the previous commit:
```
$ cat 0001-fix-imports.patch
From 3b288e1bbbddcff5457e84f1eaa7b2d4c1e11407 Mon Sep 17 00:00:00 2001
From: Casey Clements <casey.clements@continuum.io>
Date: Wed, 10 Oct 2018 21:44:56 -0400
Subject: [PATCH] fix imports

---
 pennies/calculators/__init__.py | 9 +++++----
 1 file changed, 5 insertions(+), 4 deletions(-)

diff --git a/pennies/calculators/__init__.py b/pennies/calculators/__init__.py
index 482d915..46f4a41 100644
--- a/pennies/calculators/__init__.py
+++ b/pennies/calculators/__init__.py
@@ -1,5 +1,6 @@
 # import modules that define dispatched methods, eg presentvalue, par_rate
-import assets
-import payments
-import swaps
-import trades
\ No newline at end of file
+from . import assets
+from . import payments
+from . import swaps
+from . import trades
+
--
```
This file is then added to the recipe directory, and a reference to it added in the source:patches section of meta.yaml
```
mv src/0001-fix-imports.patch recipe
cd recipe
vi meta.yaml
```
Change the source section to point back to the original url.
```
source:
  url: https://pypi.io/packages/source/{{ name[0] }}/{{ name }}/{{ name }}-{{ version }}.tar.gz
  sha256: 5044bfd6009dc4d6d14ed8d8355d0cc4b9cb4aafad105b80b0c481f8beacdc15
  patches:
    - 0001-fix-imports.patch
```
```
git add 0001-fix-imports.patch
git commit -m "Added patch"
```

Now you can build your successfully patched recipe! It no longer points to your local src repo!

`conda build recipe`

Congratulations. You've now got a few valuable tricks up your sleeve.

