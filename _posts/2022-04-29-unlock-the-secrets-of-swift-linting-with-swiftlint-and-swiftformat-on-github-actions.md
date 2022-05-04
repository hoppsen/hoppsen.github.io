---
layout: post
title:  "Unlock the Secrets of Swift Linting with SwiftLint and SwiftFormat on GitHub Actions"
tag: [CI/CD, Optimization, Helper]
category: CI/CD
---

I admit I like to work in a clean and (consistently) structured codebase. For this reason, when working with multiple people in one codebase, a linter and/or formatter is inevitable. For a couple of years, I've been utilizing a combination of [SwiftLint](https://github.com/realm/SwiftLint) and [SwiftFormat](https://github.com/nicklockwood/SwiftFormat). Most people rely solely on either SwiftLint or SwiftFormat, but I especially like the combination of SwiftLint **AND** SwiftFormat.

## How to Get There

For any new project I am starting or joining (given that there is no other linter/formatter in place), I would introduce a [`.swiftlint.yml`](https://gist.github.com/hoppsen/08b07f362bb24c56a2177c3af551bc5f) and [`.swiftformat`](https://gist.github.com/hoppsen/1f4768d1240ee4289c361d811b266649) configuration file to the root folder of the project. You can find my default configuration files [here](https://gist.github.com/hoppsen/08b07f362bb24c56a2177c3af551bc5f) and [here](https://gist.github.com/hoppsen/1f4768d1240ee4289c361d811b266649). Adding these files would be sufficient to lint/format your codebase locally after installing SwiftLint and SwiftFormat on your machine. The latter could be achieved via [brew](https://brew.sh).

### Versioning

The downside of this type of installation is that it doesn't come with a mechanism for versioning. So the version I am installing and using could be different from the version my peers are using on their machines. Additionally, the CI/CD would either use the latest or yet another version of these tools. Which would end up in unnecessarily lint errors on the CI/CD. To overcome this I found a tool by Yonas Kolb ([GitHub](https://github.com/yonaskolb), [Twitter](https://twitter.com/yonaskolb)) called ["Mint"](https://github.com/yonaskolb/Mint). With this, you can easily lock the versions of SwiftLint and SwiftFormat with a file called `Mintfile` at the root of your project:

#### Mintfile

```plain
# Source https://github.com/yonaskolb/Mint

nicklockwood/SwiftFormat@0.49.2
realm/SwiftLint@0.46.1
```

> ⚠️   After installing Mint and adding the ``Mintfile``, you need to call SwiftLint and SwiftFormat via ``mint run swiftlint ...`` or ``mint run swiftformat ...`` respectively. Otherwise, the version is not locked via Mint.

## Making Use of It within Xcode Build Phases

The first stage of linting is within the [Build Phases of Xcode](https://developer.apple.com/documentation/xcode/customizing-the-build-phases-of-a-target). Here only SwiftLint is run; otherwise, SwiftFormat would mess with our code on every build, which would be very annoying. The following snippet runs SwiftLint on every build, which results in warnings/errors at the corresponding line of code. With this, SwiftLint warnings/errors can be detected and fixed while coding.

```shell
if [ "$CI" = 'true' ]; then
  echo 'CI environment detected. Not running SwiftLint.'
elif [ "$ACTION" = 'build' ] && [ "$CONFIGURATION" = 'Debug' ]; then
  echo "No CI environment detected. 'Build' action with 'Debug' configuration detected. Running SwiftLint..."
  export PATH="$PATH:/opt/homebrew/bin" # Adds support for Apple Silicon brew directory
  if mint which swiftlint; then
    mint run swiftlint
  else
    echo 'warning: SwiftLint not installed, download from https://github.com/realm/SwiftLint'
  fi
else
  echo 'No CI environment detected. Nevertheless, not running SwiftLint.'
fi

```

## Making Use of It within Pre-commit Hook

The next stage is running SwiftFormat while committing. If SwiftFormat detects a rule violation, it will block the commit and automatically reformat the code for us to verify the changes and commit again.

```shell
#!/bin/sh

FILES_REFORMATTED=0

echo "[PRE-COMMIT] Linting project"
GIT_DIFF=$(git diff --diff-filter=d --staged --name-only)

while read FILE; do
  #
  # PLACEHOLDER: Xcode project file sorting
  # see https://hoppsen.com/posts/sort-Xcode-project-file/#automate-the-sorting
  # 

  if [[ "$FILE" == *\.swift ]]; then
    # export PATH to find mint
    export PATH="/opt/homebrew/bin:$PATH"

    BEFORE_SWIFTFORMAT_CHECKSUM=$(cat "$FILE" | shasum)
    mint run swiftformat "${FILE}" --config .swiftformat &> /dev/null
    AFTER_SWIFTFORMAT_CHECKSUM=$(cat "$FILE" | shasum)

    if [[ "$BEFORE_SWIFTFORMAT_CHECKSUM" != "$AFTER_SWIFTFORMAT_CHECKSUM" ]]; then
      echo "[PRE-COMMIT] [Warning] $FILE reformatted automatically. Please review, stage and commit"
      ((FILES_REFORMATTED++))
    fi
  fi  
done <<< "$GIT_DIFF"

if [[ "$FILES_REFORMATTED" -gt 0 ]]; then
  exit 1
fi

exit 0

```

Read more about the placeholder and how to use pre-commit hook: [Hassle Free Merging of the Xcode Project File by Sorting It](https://hoppsen.com/posts/sort-Xcode-project-file/#automate-the-sorting).

## Making Use of It within Github Actions

Whereas the first two stages are somewhat optional and solely prevent us from running into too many CI/CD linter errors, this is the place where the magic happens. The below YAML file configures to run SwiftLint and SwiftFormat on every pull request commit, ensuring that all the code is clean and formatted as intended before being merged into the main branch.

The critical point about this snippet is that it uses our previously introduced `Mintfile`. It reads the desired version and builds it on the first run. After this initial build of one to three minutes, the time for linting goes down to ~10-15 seconds for each run, thanks to caching.

Experienced users of GitHub Actions know that running Actions on macOS takes 10x more minutes than on Ubuntu. Therefore, this linting file uses Ubuntu to save valuable CI/CD minutes.

```yaml
name: Linting

on: pull_request

jobs:
  SwiftLint:
    runs-on: ubuntu-20.04
    steps:
      - uses: actions/checkout@v2
      - name: Read version from Mintfile
        run: echo "SWIFTLINT_VERSION=$(awk '/SwiftLint/ {split($0, a, "@"); print a[2]}' Mintfile)" >> $GITHUB_ENV
      - uses: actions/cache@v2
        with:
          path: .build/release/swiftlint
          key: ${{ runner.os }}-linting-swiftlint-${{ env.SWIFTLINT_VERSION }}
      - name: Build realm/SwiftLint
        run: |
          if [ -f ".build/release/swiftlint" ]; then
            sudo cp -f .build/release/swiftlint /usr/local/bin/swiftlint
          else
            git clone --depth 1 --branch ${{ env.SWIFTLINT_VERSION }} https://github.com/realm/SwiftLint
            cd SwiftLint
            swift build --disable-sandbox -c release
            mv .build .. && cd ..
            rm -rf SwiftLint
            sudo cp -f .build/release/swiftlint /usr/local/bin/swiftlint
          fi
      - name: SwiftLint
        run: |
          swiftlint --version
          swiftlint lint --strict --config .swiftlint.yml --quiet
        env:
          LINUX_SOURCEKIT_LIB_PATH: /usr/share/swift/usr/lib # FIXED: Fatal error: Loading libsourcekitdInProc.so failed
  SwiftFormat:
    runs-on: ubuntu-20.04
    steps:
      - uses: actions/checkout@v2
      - name: Read version from Mintfile
        run: echo "SWIFTFORMAT_VERSION=$(awk '/SwiftFormat/ {split($0, a, "@"); print a[2]}' Mintfile)" >> $GITHUB_ENV
      - uses: actions/cache@v2
        with:
          path: .build/release/swiftformat
          key: ${{ runner.os }}-linting-swiftformat-${{ env.SWIFTFORMAT_VERSION }}
      - name: Build nicklockwood/SwiftFormat
        run: |
          if [ -f ".build/release/swiftformat" ]; then
            if ! [ -x "$(command -v swift-format)" ]; then
              sudo cp -f .build/release/swiftformat /usr/local/bin/swiftformat
            fi
          else
            git clone --depth 1 --branch ${{ env.SWIFTFORMAT_VERSION }} https://github.com/nicklockwood/SwiftFormat
            cd SwiftFormat
            swift build --disable-sandbox -c release
            mv .build .. && cd ..
            rm -rf SwiftFormat
            sudo cp -f .build/release/swiftformat /usr/local/bin/swiftformat
          fi
      - name: SwiftFormat
        run: |
          swiftformat --version
          swiftformat . --config .swiftformat --lint
```

## Conclusion

Linting is a common practice when building apps with multiple people. SwiftLint and SwiftFormat, in the various stages, play very well together to get a more readable codebase in the project.

Feel free to contact me or tweet me on [Twitter](https://twitter.com/hoppsen1) if you have any additional tips or feedback.

Thanks!
