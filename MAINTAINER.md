# Maintainer Guide 

This guide is written for Garden Linux maintainers, explaining the steps required for creating a package for Garden Linux.
Along the series of steps, we will introduce rules in the correct context, which are outlined in the table presented in the [Overview of Rules](#overview-of-rules).

> [!NOTE]
> The overall Garden Linux release process is described in a Garden Linux Maintainer internal document [here](https://github.com/gardenlinux/process/blob/main/release.md).


## Overview of Rules
| Rule Number | Description                                             |
|-------------|-------------------------------------------------------|
| [Rule 1](#rule-1-package-git-repositories-must-be-named-accordingly) | Package git repositories must be named accordingly     |
| [Rule 2](#rule-2-git-branches-of-package-repositories-must-be-named-accordingly) | Git branches of package repositories must be named accordingly |
| [Rule 3](#rule-3-one-shot-build-dependency-repositories-must-start-with-bp-package) | One-shot build dependency repositories must start with bp-package |
| [Rule 4](#rule-4-for-existing-debian-packages-get-debian-folder-from-garden-linux-snapshot-apt-repo) | For existing debian packages, get debian folder from Garden Linux snapshot apt repo |
| [Rule 5](#rule-5-create-debian-folder-in-package-repo-only-if-debian-package-does-not-exist) | Create `debian/` folder in package repo only if Debian package does not exist |
| [Rule 6](#rule-6-get-upstream-source-from-upstream-git) | Get upstream source from upstream git                   |
| [Rule 7](#rule-7-patching-the-patches) | Patching the patches |
| [Rule 8](#rule-8-append-to-debian-patches) | Append to debian patches        |


# Use Cases 


| Use Case                           | Description                                      |
|-------------------------------------|------------------------------------------------|
| [Use Case 1](#use-case-1-build-package-with-garden-linux-pipelines) | Build package with Garden Linux pipelines       |
| [Use Case 2](#use-case-2-build-backport-package-with-garden-linux-pipelines) | Build Backport package with Garden Linux pipelines |
| [Use Case 3](#use-case-3-create-build-dependency-package-with-garden-linux-pipelines) | Create build dependency package with Garden Linux pipelines |
| [Use Case 4](#use-case-4-local-builds) | Local builds                                    |

## Use Case 1: Build package with Garden Linux pipelines
How to do a regular package build is described [here](https://github.com/gardenlinux/package-build/blob/main/README.md#github-action-build).

#### Rule 1: Package git repositories must be named accordingly
```
package-<package-source>
```
- must start with `package-` 
- `<package-source>` must be the name of the source package as it is defined in debian
   - exception: if package does not exist in debian 

## Use Case 2: Build Backport package with Garden Linux pipelines
How to do a backport package build is described [here](https://github.com/gardenlinux/package-build/blob/main/README.md#patch-releases--backporting).

#### Rule 2: Git branches of package repositories must be named accordingly

Branch types: 
- `main`: builds against latest Garden Linux environment
- `rel-<MAJOR>`: builds against `<MAJOR>` version of Garden Linux. 
- `fix/*`, `feat/*`, `other/*`: are allowed to indicate that the branch is used temporarily for work in progress   

## Use Case 3: Create build dependency package with Garden Linux pipelines

In the case when a package `ABC` requires a build dependency `XYZ` in a certain version or with a certain patch applied, we use the bp-package repositories,
which are one-shot build dependency packages. 

Those packages are NOT included in the package-releases file of gardenlinux/repo, but they are included in the package build of `ABC` with the [build_dep](https://github.com/gardenlinux/package-build/blob/290959d6fc5ba4f8c378ef931f66e7bed2b134b4/.github/workflows/build.yml#L10) argument like this:

```
on:
  push:
  workflow_dispatch:
  schedule:
    - cron: '0 0 * * *'
jobs:
  build:
    uses: gardenlinux/package-build/.github/workflows/build.yml@main
    with:
      release: ${{ github.ref == 'refs/heads/main' }}
      build_dep: gardenlinux/package-XYZ 0.0.1-0gl0+bpwhatever
```

#### Rule 3: One-shot build dependency repositories must start with bp-package
Package repositories only required as a dependency for a backported package must start with bp-package-*.

## Use Case 4: Local builds
Building packages locally is explained [here](https://github.com/gardenlinux/package-build/blob/main/README.md#local-package-build), no rules apply.
 

# Package build process 

The package build process is described in further detail in this chapter in addition to the short guide in the [README.md](https://github.com/gardenlinux/package-build/blob/main/README.md).


Overview of steps to make a package for Garden Linux:
```
1. prepare the package source
2. make source package
3. make binary package(s)
4. test binary package(s)
5. handover to repo build 
```

We will walk through each step in detail below.



## Prepare the package Source

The source to create a package consists of three parts.
1. The **upstream source code**
2. The **debian/ folder**
3. **Garden Linux patches**

It is the task of the Garden Linux package maintainer to define in the `prepare_source` script where to get upstream source, debian folder and what patches to add on top. 
In the following we will go through each of the three mentioned parts, and how to assemble them.

Since prepare_source is invoked directly in the package-build pipeline as step in [gardenlinux/package-build:bin/source](https://github.com/gardenlinux/package-build/blob/main/bin/source), we have access to multple helper functions inside the prepare_source script. Those helper scripts are introduced in context below.

### Get debian/ folder

The debian folder contains patches, configurations and rules to make and install the software. For more details about the required content of that debian folder, please read [debian documentation](https://www.debian.org/doc/manuals/maint-guide/dreq.en.html).

To create a Garden Linux package, we also need a **debian/ folder** including all the required files.
In the following we define how we **SHOULD** get the debian folder, depending on the case:


| Case | Recommended way to get debian folder | Debian Security Tracking? | Example | 
| ---- | ----------- | -------------------------------------------------- | ------- |
| Package not available in debian at all | manually maintain the **debian/ folder**. | not possible | metalbond |
| Package available in debian testing | get debian folder from out apt snapshot | yes | systemd |
| Package not available in debian testing, but in salsa | get debian folder from salsa | yes | our selected linux LTS |


#### Rule 4: Recommended way of getting debian folder is from snapshot apt-src repo
Get debian/ Folder from those snapshots, as described below

The helper script [apt_src](https://github.com/gardenlinux/package-build/blob/621c4c8f530a93884f7b9a4dfc348a50a2d19aa5/bin/source#L31C1-L31C8) must be used in prepare_source like this:
```
apt_src --ignore_orig <source_package_name> 
```
> [!NOTE]
> All Debian packages from testing are mirrored daily in a Garden Linux snapshot apt repository, if a source package is missing in a snapshot, salsa may be used to get a debian/ folder. 

#### Rule 5: For packages not existing in debian at all, create `debian/` folder in package- git repo

If there does **NOT** exist a debian package, we must define the **debian/ folder** ourself and check it in our `package-` repository. 


### Get upstream source  

We **MUST** watch upstream git repository automatically, and automatically trigger pipelines to build and test new upstream versions without waiting for a debian maintainer to upgrade salsa. 
For that, we use a scan tooling based on debian's uscan. Rules for this tool are defined per package in `debian/watch`, and include what target upstream repo to watch, and what to watch including what semversion changes should be pulled (e.g. only patchlevel).

> [!WARNING]
> Guide on how to define these `debian/watch` rules is to be done!

#### Rule 6: Get upstream source from upstream git
To enable automatic upstream version tracking, get source from upstream git repository. 
This means do NOT use apt source packages, do NOT use patches to update to a version.

```
version_orig=1234
git_src --branch <BRANCH NAME> <GIT_REPO_URL>
```

### Garden Linux Patches 

Garden Linux Patches are applied on top of debian patches. 

> [!WARNING]
> please see patching guide --- insert link here --- 

#### Rule 7: Patching debian patches 
We consider the debian/patches folder as source, and changes to debian/patches are done and tracked via patches. 
Patches for debian folder are put in folder `fixes_debian` and applied with helper function `apply_patches`.

<details>
  <summary>brief guide on how to create a Patch with package-build tools</summary>

```
package-build/build --leave-artifacts --source-only package-XYZ
package-build/build --edit package-XYZ
mkdir -p ../fixes_debian
cd run_<DATE of RUN above>
pushd b 
# ... Do your changes, recommending quilt
popd
# create your patch
diff -Naur a/debian b/debian > ../../fixes_debian/your-changes-and-fixes.patch 
# append your patch to series file
echo "your-changes-and-fixes" >> ../../fixes_debian/series
```
</details>



#### Rule 8: Append to `debian/patches`
Patches for upstream source are put in folder `upstream_patches` and applied with helper function `import_upstream_patches#`
This helper function copies the patches to debian/patches and appends them to debian/patches/series 

> [!NOTE]
> This allows us to conviniently import and maintain patches from upstream (e.g. cherry-pick an upstream commit on a different branch that fixes a CVE).

# Make source package 
A source package contains all the necessary files to build the binaries, and will be used as input by the next step [Make binary package](##Make-binary-package). 

A definition of a debian source package can be found [here](https://wiki.debian.org/Packaging/SourcePackage).


The central gardenlinux/package-build repo contains reusable actions, that are used to automatically perform this step. For reference, see that reusable action [here](https://github.com/gardenlinux/package-build/blob/621c4c8f530a93884f7b9a4dfc348a50a2d19aa5/.github/workflows/build.yml#L29)


# Make binary package 

Input for this stage is a debian source package created by the previous stage.
The central gardenlinux/package-build repo contains reusable actions, that are used to automatically perform this step, as well. 

You can add a `prepare_binary` script to the package-XYZ repo to additionally perform some preparations before the binary build step is executed. 
You could for example reconfigure debian build profiles, install build dependencies or add some logging flags for debugging.

# Test binary package 


Tests are currently disabled by default via the `nocheck` debian build profile.

From [Debian wiki](https://wiki.debian.org/BuildProfileSpec#The_DEB_BUILD_PROFILES_environment_variable)
> nocheck: No test suite should be run, and build dependencies used only for that purpose should be ignored. Builds that set this profile must also add nocheck to DEB_BUILD_OPTIONS


# Handover to repo build 
The gardenlinux/repo is responsible for pulling all required packages and create the apt repositories.  
Packages are either collected from a debian mirror, or from the respective GitHub release page in case we build the package with Garden Linux package-build pipelines.

## Daily release 
A daily scheduled gardenlinux/repo action collects all latest releases and creates a new apt repository based on latest packages. 

1. Get list of `gardenlinux/package-*` repositories
2. Download each package/version from GitHub Releases of respective package-* git repo.


## Patch release 
Packages for patch releases are also uploaded to the GitHub Release page, but must be manually selected in the gardenlinux/repo to be included for a certain patch release apt repository.  

## Disable a package-XYZ repository (NULL release handling)
If a package-XYZ build must be excluded in next nightly apt repositories, a so called "NULL release" must be published for package-XYZ. 

### What happens under the hood
The [fetch_releases](https://github.com/gardenlinux/repo/blob/main/fetch_releases) script runs as part of the github actions of gardenlinux/repo. 
`fetch_releases` goes through all repositories in gardenlinux org, and gets the latest release tag. 

The [download_pkgs](https://github.com/gardenlinux/repo/blob/ce6205aabdabd8e578c963bf230aee5e91beeb5e/download_pkgs#L27) downloads all latest release files from all the package-* GitHub Releases.
If the latest release of a given package-* contains  `null` file, it is ignored and not included in the apt repository. 

### How to do a null release

```
git checkout --detach
git commit --allow-empty -m "ignore package for repo import" 
git tag null
git push origin null
echo "" > null
gh release create null ./null --title "Null release" --notes "null"
```


If package-XYZ already has a null release but is not set as latest, you can set it manually as latest again to achieve the same result of gardenlinux/repo ignoring package-XYZ for next nightly apt repo.



