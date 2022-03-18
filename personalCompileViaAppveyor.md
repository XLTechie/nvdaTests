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

Copyright: &copy; 2022, Luke Davis and NV Access, all rights reserved.

### Introduction

For various reasons, you may not be able to, or may not wish to, construct the recommended build environment for NVDA. However, you still might want to create, build, and test your own modifications to the NVDA codebase.
Cloud-hosted services are available to build software for those not wanting to build it locally. [GitHub Actions](https://github.com/features/actions), is one example.

Another example is [Appveyor], which is the service that NV Access itself uses to build NVDA's production versions, alphas, and other public releases. It is [Appveyor] that we will explore in this article.

#### Structure of this document

In this document, all major sections are titled at heading level 3. Subsections are indicated at heading level 4, and subsidiary items are given heading level 5.

In particular: each heading for an area to incorporate into your appveyor.yml file, is at level 4, and the build scripts you need to change are at level 5.

That said, the best way to use this document, is to start at the beginning, and work your way through to the end, at least the first time you create an Appveyor build setup for NVDA.

### Prerequisites

Before following the steps in this article, you will need a [GitHub] account, and an [Appveyor] account. If you don't yet have an Appveyor account, it is possible for you to login to Appveyor with your GitHub credentials. When you go to create your account on Appveyor, choose "log in with GitHub" as your sign in method. This is not required to complete any aspect of this how-to, but it can be convenient to connect those two accounts.

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

*Note: the gist method is documented [here](https://www.appveyor.com/docs/build-configuration/#alternative-yaml-file-location). However, gist creation is slightly difficult from an accessibility prospective, although it can be done. A larger reason not to use it, is that some of NVDA's build scripts will need to be modified in-place. That is more easily done with a branch off of the NVDA fork, as described below.*

The most straight forward place to locate your build configuration file, is in the NVDA fork repository itself. However, this presents a problem. NV Access already includes an appveyor.yml file, to control their version of the build process. If you change that file, it will leak into your future pull requests, which is undesirable.

What you can do instead, is keep all build related files in a build specific branch of your my_nvda repository, and deliver appveyor.yml to Appveyor as if it was being stored on a web server other than GitHub.

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
git submodule update --init
git checkout myBuild
git rebase master
git push -f
```
But this isn't something you should do right now.

### Editing appveyor.yml

Appveyor.yml is divided into sections. [Appveyor] has quite comprehensive [documentation](https://www.appveyor.com/docs/appveyor-yml/) about their [YAML] format, and the Appveyor build configuration specifically. The many possibilities are out of scope for this document, but the minimal necessary configuration is given below.

The next several subsections will explain how to edit appveyor.yml for your own builds.
**Checkout your new build branch**, open the "`appveyor.yml`" file in an editor, and follow along.

If this is your first edit of a [YAML] file, be aware that like Python, the spacing and indentation is very important. Unlike Python, however, tabs are not the preferred indentation character, spaces are. The indentation generally progresses by one space further inward, for each keyword that modifies the starting keyword. (Technically, [Appveyor] wants two spaces per level of indentation according to their [spec](https://www.appveyor.com/docs/appveyor-yml/), but [NV Access] only uses one, so that's what we'll use here.).

The YAML file consists of several sections, each starting with a keyword, followed by a colon. The section may only hold one value or element, in which case it is usually written on the same line as the introductory keyword. For more complex sections, the initial keyword is on a line of its own, and the section that contains its value appears on the following lines. At the end of each such multi-line section, there is usually a blank line left to aid in readability. You can read more about the [YAML] format, though you shouldn't need to understand it in great detail if you are primarily copying and pasting from this how-to.

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

You will, however, need to add one variable of your own, in order to fetch any customized build scripts from your myBuild (or whatever you called it) branch.

```yaml
environment:
 PY_PYTHON: 3.7-32
 my_buildBranch: myBuild
```

* Replace "myBuild" with the name of your build branch (created above), if different than "myBuild".

#### Build environment installation:

The "install" section sets up the environment on which the application (NVDA, in this case) is built.
At this point, we start to run into scripts and auxiliary files, which NV Access uses to make the Appveyor process more modular, but which may make it a bit more tricky for our purposes.
The immediate problem we have, is that the scripts in our build branch (myBuild), are not the ones currently installed in the build environment. The scripts that are there, come from the branch you are trying to build, not the branch that configures your build.

To explain that another way: after your Appveyor configuration is finished and working, you will start editing NVDA code, which you will want to compile. For example you may have edits in the "testCode" branch. So you push the testCode branch to GitHub. Appveyor starts up, and attempts to compile that branch. That branch is checked out into your Appveyor build environment.
At this point, the build scripts that it is using, are the ones in testCode. However you aren't working on build scripts, you're working on some other part of the NVDA codebase, so the build scripts are the default ones from the NVDA master branch, which is as it should be.
But the build scripts you have modified (we will be doing that below), are in your build branch (myBuild). We must somehow make Appveyor go out and get those scripts, to use instead of the ones it already has.

We do that in the "install" section. Below, we give the current contents of that section, line by line, with discussion for each. The revised portion appears at the end.

##### Line 1, and some additions:

```yaml
install:
```

This initiates the section. The very first thing we have to do, is download our replacement build scripts from our build branch. Add the following lines:

```yaml
 - cd appveyor\scripts
 - curl -fsSL --remote-name-all "https://raw.githubusercontent.com/%APPVEYOR_REPO_NAME%/%my_buildBranch%/appveyor/scripts/{installNVDA,pushPackagingInfo,setBuildVersionVars,setSconsArgs,uploadArtifacts}.ps1"
 - cd tests
 - curl -fsSL --remote-name-all "https://raw.githubusercontent.com/%APPVEYOR_REPO_NAME%/%my_buildBranch%/appveyor/scripts/tests/{beforeTests,checkTestFailure,lintCheck,systemTests,translationCheck,unitTests}.ps1"
 - cd %APPVEYOR_BUILD_FOLDER%
```
Note that those lines are long, so be careful with line wrapping. There are only five lines shown here.

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

##### The complete install section:

Here's what the final revised install section should look like.
```yaml
install:
 - cd appveyor\scripts
 - curl -fsSL --remote-name-all "https://raw.githubusercontent.com/%APPVEYOR_REPO_NAME%/%my_buildBranch%/appveyor/scripts/{installNVDA,pushPackagingInfo,setBuildVersionVars,setSconsArgs,uploadArtifacts}.ps1"
 - cd tests
 - curl -fsSL --remote-name-all "https://raw.githubusercontent.com/%APPVEYOR_REPO_NAME%/%my_buildBranch%/appveyor/scripts/tests/{beforeTests,checkTestFailure,lintCheck,systemTests,translationCheck,unitTests}.ps1"
 - cd %APPVEYOR_BUILD_FOLDER%
 - ps: appveyor\scripts\setBuildVersionVars.ps1
 #- ps: appveyor\scripts\decryptFilesForSigning.ps1
 - py -m pip install --upgrade --no-warn-script-location pip
 - git submodule update --init
```

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

##### Edits required in appveyor\scripts\setSconsArgs.ps1:

In the `appveyor\scripts\setSconsArgs.ps1` file, there is a line (line 13, at the time of this writing), which appears as follows:
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
git add appveyor\scripts\setSconsArgs.ps1
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

#### Artifacts:

The next section identifies the artifacts our earlier sections generated. Keep this unchanged.

```yaml
artifacts:
 - path: output\*
 - path: output\*\*
```

#### Delete the deploy script:

There is a section called "deploy_script", which attempts to upload artifacts and information to the [NV Access] server, and to upload symbols to Mozilla. Since you probably don't want to do either of those things, delete the entire "`deploy_script`" section. We may cover symbols in a future version of this article, if readers are interested.

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

It is also usually advisable to set your initial clone depth to 1. There are parts of the build process that need a deeper clone, but they know how to get it.
```yaml
clone_depth: 1
```

There are many more things you can do to make the [Appveyor] based development process conform to your development preferences. This includes custom messages in the log at various stages of the build and testing process, emails sent on certain conditions or outcomes, build artifacts uploaded to private FTP servers or other locations, and much much more. All of that is beyond the scope of this document, but reading the extensive [Appveyor] documentation can give you plenty of ideas about what you can do, and how to do it.

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

* If you created an account with an email address and password, go to the [Appveyor] sign-in page, and enter them.
* If you used a developer account, such as your [GitHub] account, choose that provider from the list on the sign-in page, and follow the prompts.
* If you have recently logged in from the same browser, you shouldn't have to log in again.

1. The default page for Appveyor, once you have logged in, is the projects page. If that's not where you ended up, find and go to the projects page.
2. Find and select "NEW PROJECT".
3. If you have not already authorized Appveyor as a Github app, select "GitHub" under the "Cloud" heading, and do so now. [FixMe: more instructions needed here.]
4. Move to the "Select repository for your new project" heading, and then to the heading representing your GitHub username.
5. Below that heading, you should find your GitHub repositories listed. Go to the one representing your NVDA fork.
6. This part is less than perfectly accessible via the usual methods: you will need to use NVDA+numpad divide, to route the mouse to the repository name, and then numpad divide to click on it.
7. When you have selected the repository name, an "Add" link will appear below it. Select that link normally, and you will have attached the repository to Appveyor.

### Configuring Appveyor to build the project

After properly executing the previous section, you will be on the Appveyor project dashboard, for your NVDA fork repository.

1. Find and select the "Settings" link (last item on the last list).
2. Change the following settings:
    * "Custom configuration .yml file name": you must set this to the direct web access to your raw appveyor.yml file. Replace the relevant capitalized place holders in this URL, and enter it there: `https://raw.githubusercontent.com/GITHUB_USER_NAME/REPO_NAME/BUILD_BRANCH/appveyor.yml`
    * Find the checkbox for "Rolling builds". You probably want to select this. [Read more](https://www.appveyor.com/docs/build-configuration/#rolling-builds)
    * You may want to select the checkbox for "Do not build on pull request events.", depending on your workflow. As noted earlier, this how-to is not optimized for pull request based workflows.
3. Press the "Save" button at the bottom. There is no way with NVDA to determine that the save was successful, you can press it more than once with no ill effect, or just assume it worked.

### Try your first build

It's now time to try your first build!
You can either push something to a branch (other than master or myBuild) on your my_nvda repo, or you can select "NEW BUILD" on the Appveyor control page. If you do the latter, it will build master.

The build process is likely to take approximately 30 minutes.
You can observe the build in progress, under the "Console" section of the Appveyor interface.

You will have the option to cancel the build if something seems to be going wrong, or you don't like the branch that is being built.

Otherwise, when the build finishes, you can find the executable installer and other files (including lint and other test results) in the "Artifacts" section.

Tip: the list (l/L, comma and shift-comma), and list item (i/I) NVDA quick nav commands, can be very helpful in speeding up navigation of the Appveyor build related pages, if you learn the page layout.

### A note from the author

I hope you have found this how-to useful. If you have, I would appreciate any feedback, whether you're just saying that it worked for you, or suggestions for possible improvements, corrections, etc..

I can be reached via email at: `XLTechie+github@newanswertech.com`.
