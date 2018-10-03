# Using patch files to modify source code to get conda packages to build for your target environment

Frequently, when you try to build a recipe, it fails. This can be for any number of reasons. There are many different approaches to fixing these failures. Each depends on your particular use case.
In this post, we solve the problem by applying a patch file. This change the source code during the build, such that the built library contains your change, while the source repo (from github say) remains unchanged.

I will use an example that I encountered when I first began to build packages with C++ dependencies. The package that I was trying to build was flatbuffers, version 1.6.0 on Linux.

### For reference

- https://github.com/ContinuumIO/customer_conda_recipes/tree/master/rake)
- https://github.com/ilastik/ilastik/wiki/Creating-patches-for-Conda-recipes

Steps:

1. Start with the conda recipe. This may be from cloning the conda-forge recipe, using conda skeleton on a pypi package, or perhaps from scratch.

 There are numerous ways to do this. With a maintained library with releases, one could clone and then checkout a tag. Here we mimic this workflow by checking out a commit we treat as a release. We do this to pick out a bug that needs to be patched to get conda to build the recipe.

git clone https://github.com/fishkao/rake.git; git checkout tags/<desired version>
wget https://pypi.python.org/packages/b6/ea/f393358d1a13edaf71b38f6c70e6c89c3a1f522f70eab8d7aa2e1daf8cc9/rake-1.0.tar.gz

conda skeleton pypi rake

2. Check meta.yaml for version
3. Update recipe for your version
  - One can either examine the git log, then checkout a commit that suits or...
  - https://github.com/google/flatbuffers/archive/v1.6.0.tar.gz

4. Attempt to build: conda build recipe

5. Read logs


3. Update recipe to update dependencies 

	a) conda_build.exceptions.DependencyNeedsBuildingError: Unsatisfiable dependencies for platform osx-64: {"pbp.skels[version='>=0.2.4']", "pypirc[version='>=1.0.4']"}

	==> recipe issue. remove dependencies that do not have conda packages. 
	!!! Does this mean that conda can't handle dependencies?
	!!! An environment does not contain any pip packages (pip freeze)
```
error: Namespace package problem: rake is a namespace package, but its __init__.py does not call declare_namespace()! Please fix it.
(See the setuptools manual under "Namespace Packages" for details.)
```

	It is at this point that we begin patching.

Download the source. Trick: Use conda-render To see what the resolved source url is. `conda render r1-dependency-needs-building-error`

This will print the true source url. In our case, export URL=https://pypi.io/packages/source/r/rake/rake-1.0.tar.gz'

6. Download source	 

	a. wget in our case; tar -xvf rake-1.0.tar.gz;
	b. mkdir src; git init src; cp -R rake-1.0/* src; git add src (cd reqd?); git commit -m "Initial commit of clean source"
   
7. Update recipe to point to local source. Confirm that the same error is produced.

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

10. Update recipe to refer to the .patch file, reverting source to the original url.

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


