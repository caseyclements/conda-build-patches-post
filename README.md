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

You will need conda, conda-build, and git. conda can be downloaded [here](https://conda.io/miniconda.html). Once conda is setup, `conda install conda-build git`. 

##### 1. Our starting point - a skeleton recipe

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

##### 2. Update recipe to fix dependencies 

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

###### 3. Download source code. Create a new git repo.

Once one begins playing with the source code, it is best to create a local git repo, and make frequent commits while experimenting with building the recipe. Most likely, you will end up making changes in both the source code and the recipe.
 
![][tip] Use conda-render to see what the recipe's resolved url is. `conda render recipe`

2. Download the source
```
wget https://pypi.io/packages/source/r/rake/rake-1.0.tar.gz
tar -xvf rake-1.0.tar.gz
```
3. Initialize a repo to track source changes
```
mkdir src
git init src
cp -R rake-1.0/* src
cd src
git add .
git commit -m "Original source"
```

##### Update recipe to point to local source. Confirm that the same error is produced.



##### Fix the namespace bug. Create patch

  

8. Change source, (optionally commit) build. repeat until successful.

vi setup.py. Remove row declaring `namespace_packages=['rake'],` in `setup()` call


9. Create a .patch file from git diff. Or git show. This has the additional benefit of containing a message at the top of the patch file, which is the commit message we used.

```
git add setup.py
git commit -m "Remove incorrect namespace declaration in setup"
git show HEAD
git show HEAD > namespace.patch
cd ..
mv src/namespace.patch recipe
```

10. Update meta.yaml to refer to the .patch file, reverting source to the original url.

```
source:
  url: https://pypi.io/packages/source/{{ name[0] }}/{{ name }}/{{ name }}-{{ version }}.tar.gz
  sha256: ea169d83e06cce58fa2257dd233ef36e16aa4ad40f29413f16c62ebc94a3e2d6
  patches:
    - namespace.patch
```

11. Build again.

```
file.write('Summary: %s\n' % self.get_description())
TypeError: not all arguments converted during string formatting 
``` 
===============

Use Case #2
- python 3


