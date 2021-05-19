---
layout: post
title:  "Hassle Free Merging of the Xcode Project File by Sorting It"
tag: Xcode
category: Xcode
---

When working with multiple developers on one Xcode project, merging the project file can quickly become a hassle. I guess we have all been there. Especially if multiple features with lots of new files were added. I usually like to sort my files and folders. Mostly folders alphabetically up top, followed by all the remaining files A to Z:

```
Extensions
├── Controls
│   ├── UIBarButtonItem+DefaultButtons.swift
│   └── UIButton+AttributedTitle.swift
├── Date
│   ├── Date+Calculations.swift
│   └── Date+Formatting.swift
├── Array+Helpers.swift
├── UIViewController+KeyboardHandling.swift
└── UserDefaults+PropertyWrapper.swift
```

Doing this manually while touching specific files is fine; I do it naturally already. The pain point comes into play when merging many changes, and other developers or I am not paying close attention while solving the merge conflicts. Or you just want to get the merge of the project file done and let Xcode figure out the issues you added to the project file. 🤓

However, it eventually ends up unsorted and me stumbling across it sorts it back again. This obviously adds a lot of noise to the already hard to grasp project file while merging. Git commits like this is the result of that:

```diff
A0B4EA95214BB8D500AD1009 /* Extensions */ = {
  isa = PBXGroup;
  children = (
-   31C5428A2473B27200C60131 /* Array+Helpers.swift */,
    17F54D05265431BF00C9C3F3 /* Controls */,
    17F54D04265431B700C9C4F3 /* Date */,
-   315B8824247802AE0071644C /* UserDefaults+PropertyWrapper.swift */,
+   31C5428A2473B27200C60131 /* Array+Helpers.swift */,
    319CA18D24ADC0A4003E1208 /* UIViewController+KeyboardHandling.swift */,
+   315B8824247802AE0071644C /* UserDefaults+PropertyWrapper.swift */,
  );
  path = Controls;
  sourceTree = "<group>";
};
```

The solution to this would be to force everybody into sorting their files. When both sides of the merge are sorted, there are fewer differences when merging. Which makes almost all merges automatic or easier to tackle.

Indeed, that is not really practical. Luckily, the developers over at Apple's WebKit got us already covered with a sorting script: [sort-Xcode-project-file](https://github.com/WebKit/WebKit/blob/main/Tools/Scripts/sort-Xcode-project-file). The only thing left for us to do is the automation of that script. 🚀

## Automate the Sorting

Of course, you could just add a ``Run Script`` with the following call ``perl /<path-to-the-script>/sort-Xcode-project-file.pl <project-name>.xcodeproj`` to your Build Phases. But I'm not a huge fan of overloading the Build Phases with a bunch of scripts. Additionally, I sort it manually anyway, and even if you won't sort it manually and Xcode moves your files around when building might distract you.

Instead, I prefer to use a plain pre-commit hook with some basic shell scripting to sort the project file whenever I commit an updated version of it, given that you have placed the Perl script under a folder called ``scripts`` in the root of your git repository. You can just copy the following snippet to your pre-commit hook under ``.git/hooks/pre-commit``, and you are all set. Don't forget to update the project name unless you name all projects ``Hoppsen`` as I do. 😜

```shell
#!/bin/sh

echo "[PRE-COMMIT] Linting project"
GIT_DIFF=$(git diff --diff-filter=d --staged --name-only)

while read FILE; do
  if [[ "$FILE" == *"Hoppsen.xcodeproj/project.pbxproj"* ]]; then
    BEFORE_PROJECT_FILE_CHECKSUM=$(cat "Hoppsen.xcodeproj/project.pbxproj" | shasum)

    perl ./scripts/sort-xcode-project-file.pl ./Hoppsen.xcodeproj

    AFTER_PROJECT_FILE_CHECKSUM=$(cat "Hoppsen.xcodeproj/project.pbxproj" | shasum)

    if [ "$BEFORE_PROJECT_FILE_CHECKSUM" != "$AFTER_PROJECT_FILE_CHECKSUM" ]; then
      echo "[PRE-COMMIT] [Warning] Project file has been sorted properly. Please review and commit again!"
      git add "Hoppsen.xcodeproj/project.pbxproj"
    fi
  fi
done <<< "$GIT_DIFF"

exit 0

```

> ⚠️   The ``.git/hooks/pre-commit`` file needs to be executable! The easiest way to gain this would be to start with one of the sample files, such as ``.git/hooks/pre-commit.sample``. You can typically find it in your project folder, remove ".sample" and you are good to go.

## Setup Script

The downside of not using a ``Run Script`` in the Build Phases would be to ensure that the pre-commit hook is used by all team members. I make sure to add another step to the already existing ``setup`` lane of one of my favorite tools, [fastlane](https://fastlane.tools). This particular setup lane does things such as symlinking Xcode templates, syncing the provisioning profiles and certificates using [match](https://docs.fastlane.tools/actions/match/), or just symlinking the git hooks.

This step takes all of my git hooks stored in my repository within a folder called ``hooks`` and symlinks them to the path of the initial hooks under ``.git/hooks``.

> ⓘ   Since git 2.9, another possibility would be to change ``core.hooksPath`` to your preferred directory.

```ruby
def setup_git
  UI.header('Step: setup_git')
  gitConfigPath = '../.gitconfig'
  hooksPath = '../../hooks'
  gitHooksPath = '../.git/hooks'

  FileUtils.mkdir_p(gitHooksPath) unless File.exists?(File.expand_path(gitHooksPath))

  sh("git config include.path #{gitConfigPath}")
  sh("ln -s -f #{hooksPath}/pre-commit #{gitHooksPath}/pre-commit")

  UI.message('Your git setup looks good ✅')
end

platform :ios do
  desc 'Run this to setup your development environment'
  desc '#### Example:'
  desc "```\nbundle exec fastlane setup\n```"
  lane :setup do |options|
    setup_git
    setup_templates

    # ...

    update_code_signing_settings(
      use_automatic_signing: false,
      path: PROJECT,
    )

    sync_code_signing(
      type: 'development',
    )
  end
end
```