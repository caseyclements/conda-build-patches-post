# Using patch files to modify source code to get conda packages to build for your target environment

If you are a developer who relies upon conda, I hope to encourage you to begin building your own packages so that your projects can be used just like all of the other packages you rely upon. The success of Anaconda rests upon the ease to search for, install, and create environments for packages while automatically managing dependencies and versions. Recipes are Conda's way instructions to build a package, and they are typically very easy to create. 

[tip]: https://openclipart.org/image/24px/svg_to_png/194429/cartoon-eyes.png

Of course, sometimes things need a little help. If a recipe fails to build, there are many ways to fix it. Usually, you have permission to change both the project's recipe, and its source code. In this post, we focus on how to create a working conda recipe when you _cannot directly_ change the source code. Say, for example, that you wish to create a recipe for a tagged version in which you've found a bug.  Along the way, we describe a few common cookbook tips to ensure your recipe works. Look for this symbol ![cookbook tip][tip]

When one cannot, or wishes not to, change the code under source control, conda-build provides the ability to specify a patch file. A [patch](https://en.wikipedia.org/wiki/Patch_(Unix))  consists of a list of changes to apply to a set of text files. These changes are applied during the build, after the source is downloaded, but before the build script runs.The built package contains your change, while the source repo (i.e. in github or pypi) remains unchanged.

Patches are particularly useful in the following scenarios.
* There is a bug in the source code.
* The build is for a new target (e.g. Centos6, or Windows with Visual Studio 2013)
* The build is for a different version of Python than the author intended

In what follows, we present a tutorial that we hope will help you with your case. Some knowledge of conda-build is assumed. For reference, the docs are [here](https://conda.io/docs/user-guide/tasks/build-packages/recipe.html)

You will need conda, conda-build, and git. conda can be downloaded [here](https://conda.io/miniconda.html). Once conda is setup, `conda install conda-build git`. The instructions are written assuming a POSIX system. If you are using Windows, consider downloading Cygwin or MSYS2 to follow along.

### 1. Our starting point - a skeleton recipe

We need a starting point for our recipe. There are three typical choices.
* Manually create a [meta.yaml](https://conda.io/docs/user-guide/tasks/build-packages/define-metadata.html) from scratch or by copying an existing one.
* Cloning an existing recipe from [conda-forge](https://github.com/conda-forge) 
* Use [conda-skeleton](https://conda.io/docs/user-guide/tutorials/build-pkgs-skeleton.html) to create a recipe from a PyPI package

We've found a simple little package on PyPI to use for this tutorial, so it is natural that we use a skeleton for our starting point. Let's make a new directory to work within called 'patched',  build our skeleton recipe, and then rename it with mv. 
```
mkdir patched
cd patched
conda skeleton pypi rake
mv rake recipe
```

To track the changes we make to our recipe, let's initialize a git repository for it.
```
git init recipe; cd recipe; git add .; git commit -m "Initial skeleton recipe"; cd ..
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
These two modules don't have conda packages. They are available on PyPI though, which one can confirm with `pip search pbp.skels`. In this circumstance, the solution is to remove the dependencies as explicit requirements and let setuptools do the work. The setup script becomes the following.

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
 
![][tip] Use conda-render to see what the recipe's resolved url is. `conda render recipe`

Download the source
```
wget https://pypi.io/packages/source/r/rake/rake-1.0.tar.gz
tar -xvf rake-1.0.tar.gz
```
Initialize a repo to track source changes
```
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

```
cd recipe
git add meta.yaml
git commit -m "Update source to point to local path"
cd ..
```

Confirm that the same exeption is thrown by `conda build recipe`.  

### Fix the namespace bug.

```
error: Namespace package problem: rake is a namespace package, but its __init__.py does not call declare_namespace()! Please fix it.
(See the setuptools manual under "Namespace Packages" for details.)
```

When one looks at the setuptools manual, one quickly find that rake is NOT a namespace package. The declaration in setup() was a mistake. From setup.py, remove the row declaring `namespace_packages=['rake'],` in `setup()` call.

To confirm that this change fixes the build, run `conda build recipe --python=2.7`.

_Why do we specifically specify python 2.7? Stay tuned_

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

If it was not obvious before, our package will fail to build on Python 3. 

```
conda build recipe python=3.6
```
... produces the following error.
```
file.write('Summary: %s\n' % self.get_description())
TypeError: not all arguments converted during string formatting 
``` 

![string issues][tip] When you see exceptions involving strings, it may be a Python 2to3 issue.

Let's try to apply a second patch, to get out recipe to build for both Python 2.7 AND 3.6.
