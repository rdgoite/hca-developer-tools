# Release Process

## Introduction

In general, the build pipeline should produce a single distribution binary (Docker image, JAR file, etc.) that is promoted from development to production. This reduces the additional overhead of having to package any of the Ingest components every time the team releases new versions to any deployment environment. This also minimises any discrepancies between deployments that may be, by some non-deterministic factor, introduced through rebuild (although ideally builds should be safe to rerun).

The release process described here will allow the team to compile and curate versions of each of the system components for deployment without having to backtrack through version control history trees. This might also eliminate the effort of having to stitch through environment branches every time the team moves from one deployment environment to another, further cutting down on some seemingly unnecessary overhead. With tagged builds, the team always has a list of software versions they can deploy wherever they need.

## Current Process

At the time of writing, a release to any deployment environment requires the team (release manager standing as overall representative and coordinator) to manage and to keep track of environment branches. This is after the fact that throughout development, systems are in place to automatically build potentially shippable software packages as the team merge their code back from feature or any isolated branches to the main development branch of the version control system. Further through the the pre-release pipeline, the merges become harder to follow as fast moving main development branch tend to advance further ahead of the stabler version of the code. In general, having to keep track of multiple environment branches when running a release process makes the operation more complex than it probably should be.

## Release Process Using Tagged Builds

* Tags for building

  The system for automatically building images should only be packaging software (Docker images) for the version of the code that is tagged for building. A tag for building images is ideally composed of a descriptive prefix (say, the branch it is based upon, preferrably the `master`) followed by a short commit id, for example `master_5ddba2b`. There is no reason the team cannot just simply use (short) commit id's, but having a descriptive prefix allows any committer to add some bits of qualification to the build tag. This could also help in making the tags more easily searchable. Then again, ideally, build tags are only ever going to be used while on the main development branch (as feature or fix branches are merged back), and commit ids are usually enough information to trace back a version of the application deployed anywhere to a version in the VCS.

  Tags for building results in a list of images built and ready to be deployed to any deployment environment. This allows the team to keep track of (partially) "potentially shippable product increment" through the build tags that also serve as some sort of time or version stamps. It is from the collection of these pre-built images where the team selects the one that passes through the QA pipeline, that will ultimately be deployed to production. In a way, it's as if the `master` branch leaves behind a trace of images that are easily identifiable among each other. Tags serve as  pins that attach to specific builds of a software component. Unlike environment branches, tags do not move, and so they are better fit for keeping track of versions, something that the team already utilises for staging and production releases. Tagged builds aim to extend this to the entirety pre-release phases, effectively throughout the course of development and production. This method is akin to rock climbing where a climber attaches runners or quickdraws to bolts on the way up as they climb further, which offers a safety measure in case they slip.

  With build tags, it becomes easier to see what exact version of any of the components has been deployed to whichever environment, compared to always setting them to `master` or `latest`, or any such scheme, which does not reflect any movement in version. Using tools like `kubectl get -o json`, with tagged builds, the team can confirm that the desired version of the component is what's deployed to the proper environment.

  This also removes a lot of the overhead needed during the the release process, which, at the time of writing, requires a good amount of human intervention. This practically rids the team of having to always merge to environment branches. Once an image is built, there is very little reason to build another one based off of essentially the same exact version of the code. It is also considered good practice to build binary only once for all deployment environments as it prevents any further introduction of uncertainty through the build process. In this system, a single image flows through the entirety of the release pipeline without the need for rebuilding.

  ```
     dev
  ---o--->
     master_5ddba2b

         integration
  ---o---o--->
         master_5ddba2b

             staging
  ---o---o---o--->
             master_5ddba2b, v1.0.rc

                 production
  ---o---o---o---o
                 master_5ddba2b, v1.0.rc, v1.0
  ```

* Changelog Worthy Merge Notes

  Another area of the release process that seems to take time and effort is the preparation of the release notes or changelogs. It usually requires the release manager to run through *all* Ingest repositories to look through the commit messages to make sense of what was done between the last release and the current one. To address this, there should be a system in place that allows most of this to be automated as much as possible.

  To achieve this, the team needs to adopt a convention whereby each merge back to the main development branch is accompanied by a changelog worthy message, a brief summary of everything that has changed. This fits rather well into the branching model where each feature or fix is developed in isolation, to be merged back to a common branch later when ready. Each feature branch captures a collection of related changes aimed at achieving a particular goal. The commits within the feature branches can be summarised into a single concise, descriptive message that should potentially be the same thing that appears in the changelog. This allows the team to devise tooling to automate (a significant portion of) the process of generating changelogs.


## Developer Tools

To aid in improving the development and release processes, here are some customised convenience utilities that the team can use.

### Git Utilities

* `git short-id`

  This command returns the short version (first 7 characters) of the commit id of the current HEAD.

* `git tag-for-build`

  The `tag-for-build` utility is meant to mark the current HEAD of the main development branch with a tag for building, in preparation for deployment. It returns the resulting tag which can be piped into another utility. Once common uses for this might be to directly push to a remote repository right after tagging:

  ```
  git tag-for-build | xargs git push origin
  ```

  It is safe to run this code multiple times to display the previously created tag for building. This is useful for when the user wants to copy the tag (example for macOS):

  ```
  git tag-for-build | pbcopy
  ```

* `git changelogs`

  This lists the messages of all the merged commit into the current working branch. Ideally, this is run while the HEAD is on the main development branch (`master`). However, this can also be run for between any 2 references.

  * `git changelogs ref` - displays the changelogs between HEAD and `ref`
  * `git changelogs now before` - displays the changelogs between `now` and `before`

### Quay.io

* Security Measures

  As part of the project wide efforts in maintaining the level security, each Docker image should be cleared of any security issues. Quay.io has a built-in tool for reporting any vulnerabilities that are introduced by the dependencies within any Docker image. As a best practice, any image that is deployed to any of the deployment environments should be checked free of any such vulnerabilities. There are other tools like Snyk that can be used as an alternative.

* Managing Tags

  With the introduction of build tags, Quay.io should be configured to only initiate builds on versions marked for building. This leaves out all branches pushed to the central version control repositories, including `master`. Quay.io is set to automatically move the default `latest` Docker image tag to the build based on the `master` branch. The team can adopt different conventions, but most of the time, it is expected that the Docker image tagged with `latest` is the latest stable release. However, as the `master` branch serves as the main development branch (i.e. unstable), automatically moving `latest` to it will have the opposite effect. Quay.io does not seem to provide a way to configure this but it allows users to move tags manually. It should be part of the production release process for the `latest` tag to be moved to the image deployed to production.
