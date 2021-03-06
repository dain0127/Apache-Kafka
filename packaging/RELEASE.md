# librdkafka release process

This guide outlines the steps needed to release a new version of librdkafka
and publish packages to channels (NuGet, Homebrew, etc,..).

Releases are done in two phases:
 * release-candidate(s) - RC1 will be the first release candidate, and any
   changes to the repository will require a new RC.
 * final release - the final release is based directly on the last RC tag
   followed by a single version-bump commit (see below).

Release tag and version format:
 * release-candidate: vA.B.C-RCn
 * final release: vA.B.C



## Write release notes

Go to https://github.com/edenhill/librdkafka/releases and create a new
release (save as draft), outlining the following sections based on the
changes since the last release:
 * What type of release (maintenance or feature release)
 * A short intro to the release, describing the type of release: maintenance
   or feature release, as well as fix or feature high-lights.
 * A section of New features, if any.
 * A section of Enhancements, if any.
 * A section of Fixes, if any.

Hint: Use ´git log --oneline vLastReleaseTag..´ to get a list of commits since
      the last release, filter and sort this list into the above categories,
      making sure the end result is meaningful to the end-user.
      Make sure to credit community contributors for their work.

Save this page as Draft until the final tag is created.

The github release asset/artifact checksums will be added later when the
final tag is pushed.


## Update protocol requests and error codes

Check out the latest version of Apache Kafka (not trunk, needs to be a released
version since protocol may change on trunk).

### Protocol request types

Generate protocol request type codes with:

    $ src/generate_proto.sh ~/src/your-kafka-dir

Cut'n'paste the new defines and strings to `rdkafka_protocol.h` and
`rdkafka_proto.h`.

### Error codes

Error codes must currently be parsed manually, open
`clients/src/main/java/org/apache/kafka/common/protocol/Errors.java`
in the Kafka source directory and update the `rd_kafka_resp_err_t` and
`RdKafka::ErrorCode` enums in `rdkafka.h` and `rdkafkacpp.h`
respectively.
Add the error strings to `rdkafka.c`.
The Kafka error strings are sometimes a bit too verbose for our taste,
so feel free to rewrite them (usually removing a couple of 'the's).

**NOTE**: Only add **new** error codes, do not alter existing ones since that
          will be a breaking API change.


## Run regression tests

**Build tests:**

    $ cd tests
    $ make -j build

**Run the full regression test suite:** (requires Linux and the trivup python package)

    $ make full


If all tests pass, carry on, otherwise identify and fix bug and start over.


## Pre-release code tasks

**Switch to the release branch which is of the format `A.B.C.x` or `A.B.x`.**

    $ git checkout -b 0.11.1.x


**Update in-code versions.**

The last octet in the version hex number is the pre-build/release-candidate
number, where 0xAABBCCff is the final release for version 0xAABBCC.
Release candidates start at 200, thus 0xAABBCCc9 is RC1, 0xAABBCCca is RC2, etc.

Change the `RD_KAFKA_VERSION` defines in both `src/rdkafka.h` and
`src-cpp/rdkafkacpp.h` to the version to build, such as 0x000b01c9
for v0.11.1-RC1, or 0x000b01ff for the final v0.11.1 release.
Update the librdkafka version in `vcpkg.json`.

   # Update defines
   $ $EDITOR src/rdkafka.h src-cpp/rdkafkacpp.h vcpkg.json

   # Reconfigure and build
   $ ./configure
   $ make

   # Check git diff for correctness
   $ git diff

   # Commit
   $ git commit -m "Version v0.11.1-RC1" src/rdkafka.h src-cpp/rdkafkacpp.h


**Create tag.**

    $ git tag v0.11.1-RC1 # for an RC
    # or for the final release:
    $ git tag v0.11.1     # for the final release


**Push branch and commit to github**

    # Dry-run first to make sure things look correct
    $ git push --dry-run origin 0.11.1.x

    # Live
    $ git push origin 0.11.1.x
**Push tags and commit to github**

    # Dry-run first to make sure things look correct.
    $ git push --dry-run --tags origin v0.11.1-RC1

    # Live
    $ git push --tags origin v0.11.1-RC1


## Creating packages

As soon as a tag is pushed the CI systems (Travis and AppVeyor) will
start their builds and eventually upload the packaging artifacts to S3.
Wait until this process is finished by monitoring the two CIs:

 * https://travis-ci.org/edenhill/librdkafka
 * https://ci.appveyor.com/project/edenhill/librdkafka


## Publish release on github

Open up the release page on github that was created above.

Run the following command to get checksums of the github release assets:

    $ packaging/tools/gh-release-checksums.py <the-tag>

It will take some time for the script to download the files, when done
paste the output to the end of the release page.

Make sure the release page looks okay, is still correct (check for new commits),
and has the correct tag, then click Publish release.


### Create NuGet package

On a Linux host with docker installed, this will also require S3 credentials
to be set up.

    $ cd packaging/nuget
    $ pip3 install -r requirements.txt  # if necessary
    $ ./release.py v0.11.1-RC1

Test the generated librdkafka.redist.0.11.1-RC1.nupkg and
then upload it to NuGet manually:

 * https://www.nuget.org/packages/manage/upload


### Create static bundle (for Go)

    $ cd packaging/nuget
    $ ./release.py --class StaticPackage v0.11.1-RC1

Follow the Go client release instructions for updating its bundled librdkafka
version based on the tar ball created here.


### Homebrew recipe update

The brew-update-pr.sh script automatically pushes a PR to homebrew-core
with a patch to update the librdkafka version of the formula.
This should only be done for final releases and not release candidates.

On a MacOSX host with homebrew installed:

    $ cd package/homebrew
    # Dry-run first to see that things are okay.
    $ ./brew-update-pr.sh v0.11.1
    # If everything looks good, do the live push:
    $ ./brew-update-pr.sh --upload v0.11.1


### Deb and RPM packaging

Debian and RPM packages are generated by Confluent packaging in a separate
process and the resulting packages are made available on Confluent's
APT and YUM repositories.

That process is outside the scope of this document.

See the Confluent docs for instructions how to access these packages:
https://docs.confluent.io/current/installation.html
