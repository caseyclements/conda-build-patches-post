# Patching Source Code to Conda Build Recipes

If you are a developer who relies upon conda, I hope to encourage you to begin building your own packages so that your projects can be used just like all of the other packages you rely upon. The success of Anaconda rests upon the ease to search for, install, and create environments for packages while automatically managing dependencies and versions. Conda Recipes are the blueprint to building packages. They contain all the dependency management information, and they are typically very easy to create. 

[tip]: https://openclipart.org/image/24px/svg_to_png/194429/cartoon-eyes.png

Of course, sometimes things need a little help. If a recipe fails to build, there are many ways to fix it. Usually, you have permission to change both the project's recipe, and its source code. In this post, we focus on how to create a working conda recipe when you _cannot directly_ change the source code. Say, for example, that you wish to create a recipe for a tagged version in which you've found a bug.  Along the way, we describe a few common cookbook tips to ensure your recipe works. Look for this symbol ![cookbook tip][tip]

When one cannot, or wishes not to, change the code under source control, conda-build provides the ability to specify a patch file. A [patch](https://en.wikipedia.org/wiki/Patch_(Unix))  consists of a list of changes to apply to a set of text files. These changes are applied during the build, after the source is downloaded, but before the build script runs.The built package contains your change, while the source repo (i.e. in github or pypi) remains unchanged.

Patches are particularly useful in the following scenarios.
* There is a bug in the source code.
* The build is for a new target (e.g. Centos6, or Windows with Visual Studio 2013)
* The build is for a different version of Python than the author intended

In what follows, we present a tutorial that we hope will help you with your case. Some knowledge of conda-build is assumed. For reference, the docs are [here](https://conda.io/docs/user-guide/tasks/build-packages/recipe.html)

You will need conda, conda-build, and git. conda can be downloaded [here](https://conda.io/miniconda.html). Once conda is set up,  use it to install build and git. `conda install conda-build git`. The instructions that follow are written assuming a POSIX system. If you are using Windows, consider downloading Cygwin or MSYS2 to follow along.

### 1. Our starting point - a skeleton recipe

We need a starting point for our recipe. There are three typical choices.
* Manually create a [meta.yaml](https://conda.io/docs/user-guide/tasks/build-packages/define-metadata.html) from scratch or by copying an existing one.
* ![conda-forge][tip] Clone an existing recipe from [conda-forge](https://github.com/conda-forge) 
* ![conda-skeleton][tip] Use [conda-skeleton](https://conda.io/docs/user-guide/tutorials/build-pkgs-skeleton.html) to create a recipe from a PyPI package

We happened to stumble upon a small package in PyPI that required patching to create a recipe for. We don't know anything about the package internals, but we will use it here. As the package is on PyPI, we will use a skeleton for our starting point.  Let's make a new directory to work within called 'patched',  build our skeleton recipe, and then rename it with mv. 
```
mkdir patched
cd patched
conda skeleton pypi rake
mv rake recipe
```

To track the changes we make to our recipe, let's initialize a git repository for it.
```
git init recipe 
cd recipe
git add .
git commit -m "Initial skeleton recipe"
cd ..
```

Finally, let's try to build our recipe.

```
conda build recipe
```

### 2. Update recipe to fix dependencies 

The build log should throw an exception like the following.
```
conda_build.exceptions.DependencyNeedsBuildingError: Unsatisfiable dependencies for platform osx-64: {"pbp.skels[version='>=0.2.4']", "pypirc[version='>=1.0.4']"}
```
![externally-managed][tip] These two modules don't have conda packages. They are available on PyPI though, which one can confirm with `pip search pbp.skels`. In this circumstance, the solution is to remove the dependencies as explicit requirements and let setuptools do the work. The setup script becomes the following.

```
script: python setup.py install --single-version-externally-managed --record record.txt
```

Make the following changes to `meta.yaml`

```
 build:
   number: 0
-  script: "{{ PYTHON }} -m pip install . --no-deps --ignore-installed -vvv "
+  script: python setup.py install --single-version-externally-managed --record record.txt

 requirements:
   host:
-    - pbp.skels >=0.2.4
-    - pip
-    - pip
-    - pypirc >=1.0.4
     - python
   run:
-    - pbp.skels >=0.2.4
-    - pypirc >=1.0.4
     - python
```

When we rebuild, `conda build recipe`, we get a new exception. 
```
error: Namespace package problem: rake is a namespace package, but its __init__.py does not call declare_namespace()! Please fix it.
(See the setuptools manual under "Namespace Packages" for details.)
```
This is an issue with the source, not the recipe. It is at this point that we begin patching.

### 3. Download source code. Create a new git repo. Point recipe to it.

Once one begins playing with the source code, it is best to create a local git repo, and make frequent commits while experimenting with building the recipe. Most likely, you will end up making changes in both the source code and the recipe.
 
![render][tip] Use conda-render to test that your recipe is consistent. You can see what the recipe's full url is, along with the entire list of required packages and their versions. `conda render recipe`

Download the source. Initialize a repo to track source changes.
```
wget https://pypi.io/packages/source/r/rake/rake-1.0.tar.gz
tar -xvf rake-1.0.tar.gz
mkdir src
git init src
cp -R rake-1.0/\* src
cd src
git add .
git commit -m "Original source"
cd ..
```

Update recipe to point to local source. Make the following changes to meta.yaml
```
 source:
-  url: https://pypi.io/packages/source/{{ name[0] }}/{{ name }}/{{ name }}-{{ version }}.tar.gz
-  sha256: ea169d83e06cce58fa2257dd233ef36e16aa4ad40f29413f16c62ebc94a3e2d6
+  #url: https://pypi.io/packages/source/{{ name[0] }}/{{ name }}/{{ name }}-{{ version }}.tar.gz
+  #sha256: ea169d83e06cce58fa2257dd233ef36e16aa4ad40f29413f16c62ebc94a3e2d6
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

### Fix the namespace bug.

```
error: Namespace package problem: rake is a namespace package, but its __init__.py does not call declare_namespace()! Please fix it.
(See the setuptools manual under "Namespace Packages" for details.)
```

When one looks at the setuptools manual, one quickly finds that rake is NOT a namespace package. It's structure is straightfoward. The namespace declaration in setup() was just a mistake. From setup.py, remove the row declaring `namespace_packages=['rake'],` in `setup()` call.

To confirm that this change fixes the build, run `conda build recipe --python=2.7`.

_Why do we specifically specify python 2.7? Stay tuned for patch number two..._

### Create a patch

Having confirmed that our change to the source code has fixed the build, we commit the change, and then use git to produce a patch file.
```
cd src
git add setup.py
git commit -m "remove namespace declaration"
git format-patch -1
```

This produces a file named '0001-remove-namespace-declaration.patch'. It shows the difference between the latest and the previous commit:
```
From c5e85f0861f1670c9f930641a2adfe75fa5c412e Mon Sep 17 00:00:00 2001
From: Casey Clements <casey.clements@continuum.io>
Date: Wed, 3 Oct 2018 23:39:37 -0400
Subject: [PATCH] remove namespace declaration

---
 setup.py | 1 -
 1 file changed, 1 deletion(-)

diff --git a/setup.py b/setup.py
index b846398..32656a8 100644
--- a/setup.py
+++ b/setup.py
@@ -15,7 +15,6 @@ setup(name= 'rake',
     url="https://github.com/fishkao/rake",
     packages = find_packages(),
     package_data = {'':['*.*'],'rake':['*.*']},
-    namespace_packages=['rake'],
     long_description=open(README).read() + "\n\n",
     description = ('Rapid Automatic Keywords Extraction', "Just a Practice"),
     classifiers=[
--
```
This file is then added to the recipe directory, and a reference to it added in the source:patches section of meta.yaml
```
mv src/0001-remove-namespace-declaration.patch recipe
cd recipe
vi meta.yaml
```
Change the source section to point back to the original url.
```
source:
   path: ../src
   patches:
     - 0001-remove-namespace-declaration.patch
```
```
git add .
git commit -m "Added namespace patch"
```

Now you can build your successfully patched recipe! It no longer points to your local src repo!

`conda build recipe --python=2.7`

### A Second Patch - Python 3

If it has not been made clear, our package will fail to build on Python 3. 

```
conda build recipe python=3.6
```
... produces the following error.
```
file.write('Summary: %s\n' % self.get_description())
TypeError: not all arguments converted during string formatting 
``` 

##### A few quick fixes
To get this to build on Python 3 will take a number of small changes to the source. I will quickly go through each of the changes made as I believe that they are common enough to merit being pointed out, although this is by no means a critique of the package. It is merely used as an example.

* The string exception was a simple type error. The package description was passed as a tuple, not a string.
```
-    description = ('Rapid Automatic Keywords Extraction', "Just a Practice"),
+    description = 'Rapid Automatic Keywords Extraction. Just a Practice',
```
* A bytecode.pyc file had been mistakenly committed.
* The requirements file included packages that are not imported. Further, they require a package that has no Python 3 support, so we removed their requirement from setup.py
```
-    install_requires=[x.replace("==",">=") for x in open(REQUIREMENT).read().split('\n') if x!=""],
```

For each of these, we make a separate commit. If we do so, we can use `git format-patch -4` to create 4 patch files, each from one commit and using the commit message to name the file. In our case, the changes above result in the following.
```
0001-remove-namespace-declaration.patch
0002-change-setup-description-from-tuple-to-string.patch
0003-remove-pyc.patch
0004-remove-unneeded-python2-requirements.patch
```

##### Basic 2to3 changes

Only when we try to run the rake module do we see the need to make patches for a Python3 packge.

Add the following test section to meta.yaml

```
test:
  imports:
    - rake.rake
```
```
conda build recipe
```
```
if debug: print keywordcandidates
                                    ^
SyntaxError: Missing parentheses in call to 'print'
```

Print statements have been deprecated. To be consistent between python 2 and 3, we replace print x with print(x).

A much more subtle difference between Python 2 and 3 is the default behavior of the division operator `/`. In Python 2, operating on numbers, it will cast the result to an int. (e.g. `5/3==1`. In Python3, however, / returns a float (e.g. 5/3 ~= 1.666). This is often difficult to catch without tests. Luckily, in this particular package, the author has used the Python 2 behavior to slice a list. In Python 2 `range(5)[:10/3]` works. On Python 3, you must cast this to an int, `range(5)[:int(10/3]`. Fortunately the latter works in both.

To get / to produce a float in Python2, `from __future__ import division`.

Applying these changes completes our patching!

```
$ git diff
diff --git a/rake/rake.py b/rake/rake.py
index 81646dc..391b11d 100755
--- a/rake/rake.py
+++ b/rake/rake.py
@@ -4,6 +4,9 @@
 # Automatic keyword extraction from indi-vidual documents.
 # In M. W. Berry and J. Kogan (Eds.), Text Mining: Applications and Theory.unknown: John Wiley and Sons, Ltd.

+from __future__ import division
+from __future__ import print_function
+
 import re
 import operator
 import math
@@ -128,7 +131,7 @@ def rake(text):

        totalKeywords = len(sortedKeywords)
        if debug: print(totalKeywords)
-       print(sortedKeywords[0:(totalKeywords/3)])
+       print(sortedKeywords[:int(totalKeywords/3)])
        return  sortedKeywords

 if test:
~/code/blog/patched/src
$ git add rake
~/code/blog/patched/src
$ git commit -m "fix division py2to3"
[master 9d4a372] fix division py2to3
 1 file changed, 4 insertions(+), 1 deletion(-)
~/code/blog/patched/src
$ git lg
* 9d4a372 - (HEAD -> master) fix division py2to3 (2018-10-04 16:58:56 -0400) <Casey Clements>
* 451f26c - print_function (2018-10-04 16:55:53 -0400) <Casey Clements>
* fb0a70f - remove unneeded python2 requirements (2018-10-04 13:36:23 -0400) <Casey Clements>
* 09ae952 - remove pyc (2018-10-04 13:34:38 -0400) <Casey Clements>
* 54eefdf - change setup description from tuple to string (2018-10-04 13:33:50 -0400) <Casey Clements>
* c5e85f0 - remove namespace declaration (2018-10-03 23:39:37 -0400) <Casey Clements>
* 77313d3 - Original source (2018-10-03 23:15:08 -0400) <Casey Clements>
~/code/blog/patched/src
$ git format-patch -6
0001-remove-namespace-declaration.patch
0002-change-setup-description-from-tuple-to-string.patch
0003-remove-pyc.patch
0004-remove-unneeded-python2-requirements.patch
0005-print_function.patch
0006-fix-division-py2to3.patch
```

Finally, copy over our 6 patches to the recipe, and update the source section. Add the patches, and change the source back from path to url. Commit this. Test that it builds on python 2.7 and 3.6, and upload the packages to anaconda.org!

Congratulations. You've now got a few valuable tricks up your sleeve.

