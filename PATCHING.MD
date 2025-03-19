# Create or fix Patches

In the following we will go through one way of creating a patch for a gardenlinux package. 
We will use the repository package-linux and the resulting gardenlinux package "linux" as an example.

#### 1. Prepare your local sources

First, we need the sources of the package-linux with all patches applied exactly as the GitHub package-build pipeline would do. 
Later on, when we have the sources, we can fix, remove, add patches to the sources easily. 


```
./package-build/build --leave-artifacts --source-only package-linux
```
This step has called the [package-build/bin/source](https://github.com/gardenlinux/package-build/blob/main/bin/source) script inside a container, and placed the sources inside the local folder `package-linux/output/run-<date-time>/a`, and another copy `package-linux/output/run-<date-time>/b` right next to it. Even if the source script fails to apply a certain patch, you will have the source folders exactly in the state where the source build left bailed out. 


> [!Warning]
> If you run this on arm64, then you need to also pass `--arch arm64` for the source build. Cross-build for generating sources is not required and might cause issues.

> [!Note]
> If the package-build/bin/source has failed, the sources are kept and are in the state where the package-build/bin/sources exited. 

#### 2. Spawn a temporary linux container to use quilt 

Quilt is a versatile tool for working with patches. You can use it to fix, add or remove patches. 
Patches inherited from debian (the patches inside the debian/patches folder that we pull) are in allmost all cases maintained with quilt.

If we want to fix a patch from debian, then we need to make sure to use the same patch format as debian. Quilt can be [configured accordingly](https://wiki.debian.org/UsingQuilt#Using_quilt_with_Debian_source_packages). 

The following command spawns a container with quilt already configured correctly.


```
./package-build/build --edit package-linux
```

This starts a debian container with the output folder generated from the previous step mounted. 

Preparations are defined in [package-build/bin/patchenv-init](https://github.com/gardenlinux/package-build/blob/main/bin/patchenv-init),
you can review those and use your local machine instead. 

#### 3. Make your changes inside folder b 
Debian guide references [this](http://www.shakthimaan.com/downloads/glv/quilt-tutorial/quilt-doc.pdf) tutorial, licensed under the GNU Free Documentation License, and may be a good starting point for you. 


#### 4. Create the patch 

The diff between folder a and your edited folder b is now the patch. We just need to create it in an acceptable format:

```
diff -Naur a/ b/ > name-of-your-patch.patch 
```

> [!Tip]
> If you already know what file has changed, e.g. because you fix a debian/patch/file, then you can instead use
> ```
> diff -Naur a/path/to/file b/path/to/file > name-of-your-patch.patch 
> ```

#### 5. Append the patch to the patches folder

Depending on the package you are working on, there may or may not already exist an appropriate patches folder where you can add the `name-of-your-patch.patch` to.

If it does not exist yet, you need to create a folder (e.g. called `fixes_debian`) and then adapt the prepare_source script to apply patches by adding for example this line
```
apply_patches fixes_debian
```

apply_patches is a bash function sourced from [package-build/bin/source](https://github.com/gardenlinux/package-build/blob/main/bin/source) that applies all patches as defined by the series file. 
See example [package-linux](https://github.com/gardenlinux/package-linux/blob/18baefb947b6fb3a4abaa9c58b6a42be3117e6dd/prepare_source#L30C1-L30C27)


