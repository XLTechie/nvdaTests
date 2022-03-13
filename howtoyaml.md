## Compiling NVDA Without a Local Build Environment Using Appveyor

[//]: # (Links for use elsewhere in the document)
[git]: https://www.git-scm.com
[GitHub]: https://www.github.com/
[NVDA]: https://github.com/nvaccess/nvda/
[NV Access]: https://www.nvaccess.org/
[Appveyor]: https://appveyor.com/
[Contributing]: https://github.com/nvaccess/nvda/wiki/Contributing
[YAML]: https://yaml.org/

Author: Luke Davis, Open Source Systems, Ltd. (@XLTechie on [GitHub])

Copyright: &copy; 2022, NV Access and Luke Davis, all rights reserved.

### Introduction

For various reasons, you may not be able to, or may not wish to, construct the recommended build environment for NVDA. However, you still might want to create, build, and test your own modifications to the NVDA codebase.
Cloud-hosted services are available to build software for those not wanting to build it locally. [GitHub Actions](https://github.com/features/actions), is one example.
Another example is [Appveyor], which is the service that NV Access itself uses to build NVDA's production versions, alphas, and other public releases. It is [Appveyor] that we will explore in this article.

### Prerequisites

Before following the steps in this article, you will need a [GitHub] account, and an [Appveyor] account.

It is assumed that you have already read the [NVDA] page on [Contributing], in particular the section on the [GitHub process](https://github.com/nvaccess/nvda/wiki/Contributing#github-process), as you will need to have some version of that already in place on your local system.

Furthermore, you should already be generally familiar with [git], [GitHub], and how to work with repositories. This article won't go into any of these basic areas in any detail, and will only summarize topics the experienced developer is expected to understand already.

### Situations not covered in this article

* This article assumes your NVDA coding workflow involves branching off of master (or another branch), working on the code, and pushing the result to your NVDA fork. We don't cover, and the modifications we make to appveyor.yml are not tested with, any workflow which involves issuing pull requests against your NVDA fork. It may work, but results are unknown.
* We don't include symbol retention and uploading in this article, and disable related elements.
* Currently, we disable elements related to code signing, and don't attempt it.

*N.B. A future version of this document may cover [Windows package self-signing](https://docs.microsoft.com/en-us/windows/msix/package/create-certificate-package-signing), so that certain Windows features which require signed copies, will work correctly. However we will not be able to follow the [NV Access] model for this, as used with their Appveyor configuration--a different procedure will need to be developed.*

### Overview

We will undertake the following general steps:
* Setup GitHub and local environments.
* Prepare local repository for Appveyor.
* Modify the Appveyor configuration already included with NVDA to work for unofficial purposes.
* Tell Appveyor where to find the new repository.
* Configure Appveyor to find the config file.
* Try the first build!

### Getting started

If you haven't already, per the "Prerequisites" section above, fork a copy of the main [NVDA] repository into your GitHub account. You can call your fork "NVDA", or "my_nvda", or whatever you want. For the rest of this article, we will assume you have called it "my_nvda" to make a clear separation from the NV Access version of the repository; though just calling it "nvda" is the normal practice.

Next, clone the fork to your local dev environment, and perform the other setup tasks recommended in the [Contributing] document.

### Appveyor.yml: the build file

[Appveyor] has several ways that it can access its configuration file, appveyor.yml. Consequently, you have several ways you can proceed at this point. One way, is to host the file completely outside your NVDA fork, such as in a [GitHub] Gist.
Instructions for those methods are not included at this time.

The most straight forward place to locate your build configuration file, is in the NVDA fork repository itself. However, this presents a problem. NV Access already includes an appveyor.yml file, to control their version of the build process. If you change that file, it will leak into your future pull requests, which is undesirable.
Fortunately, Appveyor has thought of this, and allows keeping your build file in a separate branch, so that's the method we will describe here.

#### A branch for your build file

We will make a branch that is dedicated to nothing other than our customized build process. This will enable us to store the appveyor.yml file for appveyor to load and work from, as well as any build scripts we may customize.
Change into the directory of your fork, and create and checkout a new branch based on master:
```
cd my_nvda
git checkout master
git checkout -b myBuild
```
Here, we have called the branch "myBuild" just to be clear, but you could call it "appveyor", or anything you want; the name doesn't matter, as long as it makes sense to you, and is unique within the NVDA project.

None of the files in this branch matter, except for `appveyor.yml`, and the files under the `appveyor` directory. You are never going to merge this branch with master. It exists strictly to allow you to keep your copy of `appveyor.yml`, and any customized build scripts, separate from the NV Access copy in the master branch.
If, in the future, you do want to keep the rest of it up to date with master (for example if NV Access modifies the build scripts), you can do so by rebasing on to master periodically as follows:
```
git checkout master
git pull
git submodule update
git checkout myBuild
git rebase master
git push
```
But this isn't something you should do right now.

### Editing appveyor.yml

Appveyor.yml is divided into sections. [Appveyor] has quite comprehensive [documentation](https://www.appveyor.com/docs/appveyor-yml/) about their [YAML] format, and the Appveyor build configuration specifically. The many possibilities are out of scope for this document, but the minimal necessary configuration is given below.

The next several subsections will explain how to edit appveyor.yml for your own builds.
**Checkout your new build branch**, open the "`appveyor.yml`" file in an editor, and follow along.
There is a sample file of the end result available [here](FixMe!), but it is best to read all of this, because there are some sections which must contain values specific to your setup.

If this is your first edit of a [YAML] file, be aware that like Python, the spacing and indentation is very important. Unlike Python, however, tabs are not the preferred indentation character, spaces are. The indentation generally progresses by one space further inward, for each keyword that modifies the starting keyword. (Technically, [Appveyor] wants two spaces per level of indentation according to their [spec](https://www.appveyor.com/docs/appveyor-yml/), but [NV Access] only uses one, so that's what we'll use here.).

The YAML file consists of several sections, each starting with a keyword, followed by a colon. The section may only hold one value or element, in which case it is usually written on the same line as the introductory keyword. For more complex sections, the initial keyword is on a line of its own, and the section that contains its value, appears on following lines. At the end of each such multi-line section, there is usually a blank line left to aid in readability. You can read more about the [YAML] format, though you shouldn't need to understand it in great detail if you are primarily copying and pasting from this how-to.

#### The top of the file:

The first couple lines of the file, are generally used to specify the build operating system and IDE, and the basic version string to be used by built executables.

The existing lines are fine for that.
```yaml
os: Visual Studio 2019
version: "{branch}-{build}"
```

#### Branches to build:

You will first need to edit the "branches" keyword. This tells Appveyor what branches to build. For [NV Access], it is necessary to build production releases, pull request try builds, and other things.

```yaml
branches:
 only:
  - master
  - beta
  - rc
  - /try-.*/
  - /release-.*/
```
You probably don't want to be so specific about what to build. In fact, you most likely want to build all branches that you push, except for master and the branch containing your appveyor config.
If that is what you want, then change this section, replacing the "only" keyword with the "except" keyword, and list the undesired branches:

```yaml
branches:
 except:
  - master
  - myBuild
```

#### Environment:

In the "environment" section, the only variable you need from the NV Access version of the file, is the Python version. This will be 3.7, 32 bit, unless you are experimenting with non-standard Python versions. You may comment out, delete, or leave alone, the rest of the environment variables; they pertain to symbol uploading and code signing, neither of which is covered by this how-to, or required for most personal builds.

You will, however, need to add a few of your own variables, in order to fetch any customized build scripts from your myBuild (or whatever you called it) branch.

```yaml
environment:
 PY_PYTHON: 3.7-32
 my_ghUser: GITHUB_USER
 my_repo: REPO_NAME
 my_buildBranch: myBuild
```

* Replace "GITHUB_USER" with your [GitHub] username.
* Replace "REPO_NAME" with the name of your NVDA fork, such as my_nvda.
* Replace "myBuild" with the name of your build branch (created above), if different than "myBuild".

#### Build environment installation:

The "install" section sets up the environment on which the application (NVDA, in this case) is built.
At this point, we start to run into scripts and auxiliary files, which NV Access built to make the Appveyor process easier for them, but which may make it a bit more tricky for our purposes.
The immediate problem we have, is that the scripts in our build branch (myBuild), are not the ones currently installed in the build environment. The scripts that are there, come from the branch you are trying to build, not the branch that configures your build.

To explain that another way: after your Appveyor configuration is finished and working, you will start editing NVDA code, which you will want to compile. For example you may have edits in the "testCode" branch. So you push the testCode branch to GitHub. Appveyor starts up, and attempts to compile that branch. That branch is checked out into your Appveyor build environment.
At this point, the build scripts that it is using, are the ones in testCode. However you aren't working on build scripts, you're working on some other part of the NVDA codebase, so the build scripts are the default ones from the NVDA master branch, which is as it should be.
But the build scripts you have modified (we will be doing that below), are in your build branch (myBuild). We must somehow make Appveyor go out and get those scripts, to use instead of the ones it already has.

We do that in the "install" section. Below, we give the current contents of that section, line by line, with discussion for each. The revised section appears at the end.

##### Line 1, and some additions:

```yaml
install:
```

This initiates the section. The very first thing we have to do, is download our replacement build scripts from our build branch. Add the following lines:

```yaml
 - curl -fsSL --clobber --output-dir appveyor\scripts --remote-name-all "\
   https://raw.githubusercontent.com/%my_gitUser%/%my_repo%/%my_buildBranch%/appveyor/scripts/\
   {installNVDA,pushPackagingInfo,setBuildVersionVars,setSconsArgs,uploadArtifacts}.ps1"
 - curl -fsSL --clobber --output-dir appveyor\scripts\tests --remote-name-all "\
   https://raw.githubusercontent.com/%my_gitUser%/%my_repo%/%my_buildBranch%/appveyor/scripts/tests\
   {beforeTests,checkTestFailure,lintCheck,systemTests,translationCheck,unitTests}.ps1"
```
Note that those lines are long, so be careful with line wrapping. There are only six lines shown here, although if you understand YAML line wrapping you can change the line lengths and count to fit your editing preferences.

##### Original line 2:

```yaml
 - ps: appveyor\scripts\setBuildVersionVars.ps1
```
This runs a PowerShell script, which sets the build version, and contains logic for deciding whether this is a pull request, try, alpha, beta, release, or other kind of build. In general this will result in a sane name for branches you build, so nothing should need to change here.

##### Original line 3:

```yaml
 - ps: appveyor\scripts\decryptFilesForSigning.ps1
```
We aren't signing, so you should comment this line out.
```yaml
 #- ps: appveyor\scripts\decryptFilesForSigning.ps1
```

##### Original lines 4 and 5:

```yaml
 - py -m pip install --upgrade --no-warn-script-location pip
 - git submodule update --init
```
You definitely want these lines; they prepare the Python environment, and update the git submodules which NVDA uses. After you're sure things are working, you might want to add a " -q" (quiet) flag after the "git submodule" command, to clean up your logs a little bit.

#### The build script:

This section controls the build process. We should leave it largely unchanged, although we will have to edit one of its scripts.

```yaml
build_script:
 - ps: appveyor\scripts\setSconsArgs.ps1
 - scons source %sconsArgs%
 - scons %sconsOutTargets% %sconsArgs%
 - ps: appveyor\scripts\buildSymbolStore.ps1
 - 7z a -tzip -r output\symbols.zip symbols\*.dl_ symbols\*.ex_ symbols\*.pd_
```

Note: if you do not care about symbols, you can comment out or delete those last two lines.

##### Edits required in scripts/setSconsArgs.ps1:

In the `scripts/setSconsArgs.ps1` file, there is a line (line 13, at the time of this writing), which appears as follows:
```
$sconsArgs += ' publisher="NV Access"'
```
Since you are not [NV Access], you should change this to your own name, organization, or preferred identifier.

Additionally, there are two blocks of code in this file that relate to code signing. In order to prevent errors related to this area, you need to comment these blocks of code out by placing a hash mark in the far left column of each line. The two sections you must comment out are:

```
if(!$env:APPVEYOR_PULL_REQUEST_NUMBER) {
        $sconsOutTargets += " appx"
}
```
(Which are lines three-five of the file as of this writing); and:
```
if(!$env:APPVEYOR_PULL_REQUEST_NUMBER) {
        $sconsArgs += " certFile=appveyor\authenticode.pfx certTimestampServer=http://timestamp.digicert.com"
}
```
(Which are lines 14-16 of the file as of this writing).

Don't forget to run `git add` after you modify this, or any other, script.
```
git add scripts/setSconsArgs.ps1
```

#### Test sections:

The next three sections configure and use the testing environment. We can leave these unchanged.
```yaml
before_test:
 - ps: appveyor\scripts\tests\beforeTests.ps1
 - ps: appveyor\scripts\installNVDA.ps1

test_script:
 - ps: appveyor\scripts\tests\translationCheck.ps1
 - ps: appveyor\scripts\tests\unitTests.ps1
 - ps: appveyor\scripts\tests\lintCheck.ps1
 - ps: appveyor\scripts\tests\systemTests.ps1

after_test:
 - ps: appveyor\scripts\tests\checkTestFailure.ps1
```

##### Fixme: do we need to worry about tests upload in tests/unitTests.ps1?

##### FixMe: do we need to worry about tests/lintCheck.ps1?

#### Artifacts:

The next section identifies the artifacts our earlier sections generated. Keep this unchanged.

```yaml
artifacts:
 - path: output\*
 - path: output\*\*
```

#### Delete the deploy script:

There is a section called "deploy_script", which attempts to upload artifacts and information to the [NV Access] server, and to upload symbols to Mozilla. Since you probably don't want to do either of those things, delete the entire "`deploy_script`" section. We may cover symbols in a future version of this article; ask if you want that here.

#### Files from failure:

The next section causes the upload of files (artifacts), even in the event of a failure. Since that is desirable, leave this section as-is.

```yaml
on_failure:
 - ps: appveyor\scripts\uploadArtifacts.ps1
```

#### Finishing up:

And finally, what to do when everything finishes. Again, we can leave this section intact.

```yaml
on_finish:
 - ps: appveyor\scripts\pushPackagingInfo.ps1
```

##### Suggested edit to appveyor\scripts\pushPackagingInfo.ps1:

In the "`appveyor\scripts\pushPackagingInfo.ps1`" script, there is one line that you might want to change, though it is not all that important.
```
	Add-AppveyorMessage "Build (for testing PR): $exeUrl"
```

Since generally you will be testing branches, and not pull requests, you might want to change "PR" in the above, to the word "branch".

#### Miscellaneous:

You should probably make sure that Appveyor only builds one branch at a time. This is all it can do for free accounts as of the time of this writing, but in case that ever changes, it won't hurt to include the following somewhere in your file:
```yaml
max_jobs: 1
```

There are many more things you can do to make the [Appveyor] based development process conform to your development preferences. This includes custom messages in the log at various stages of the build and testing process, emails sent on certain conditions or outcomes, build artifacts uploaded to private FTP servers or other locations, and much much more. All of that is beyond the scope of this document, but reading the extensive [Appveyor] documentation can be give you ideas about what you can do, and how to do it.

### When your files are ready: commit and push

Once your appveyor.yml file is ready, be sure to add it to a commit:
```
git add appveyor.yml
```

If you have modified any scripts, as you should have done if you were following this how-to, make sure you have also added those to a commit.

Next you need to run:
```
git commit
```
or:
```
git commit -a
```
(To have git automatically find modified files without having to run `git add`), enter a commit message describing what you just did, and finish the commit.
Now you will want to push your branch to GitHub.
```
git push -u origin myBuild
```

### Setting up the project on Appveyor

It's now time to log in to Appveyor, and configure the project.
After logging in, go to: 

