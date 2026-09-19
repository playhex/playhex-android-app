# Android app

App id: `org.playhex.twa`

This repo contains source code for [PlayHex](https://playhex.org) TWA.
It is needed at least for F-Droid to build apk and add it in its store.

## Build locally (Google Play)

Builds and signs the `.aab` to upload on Google Play Console.

https://hub.docker.com/r/mingc/android-build-box/

### 1. Update version

In `app/build.gradle`:

```
        versionCode 2026091900  # date+hour: "2026091900" for 19 sept 2026, 00h
        versionName "1.0.0.10"  # next "1.0.0.X"
```

Add the changelog for this version code in every language, e.g `fastlane/metadata/android/fr/changelogs/2026091900.txt`.

Then commit and push, and tag with the version code (F-Droid builds from that tag).

### 2. Build the aab

Requires `signing.keystore` in the current folder (from my vault, "playhex key").

Note: `--dns` is needed when the host DNS is not reachable from inside the container
(otherwise gradle fails with `UnknownHostException: services.gradle.org`).

``` bash
# build aab -> app/build/outputs/bundle/release/app-release.aab
docker run --rm --dns 1.1.1.1 --dns 8.8.8.8 -v `pwd`:/project mingc/android-build-box \
  bash -c "cd /project && ./gradlew build && ./gradlew app:bundleRelease"

cp app/build/outputs/bundle/release/app-release.aab .

# sign aab (asks for the keystore password, see signing-key-info.txt)
docker run -it --rm -v `pwd`:/project -w /project mingc/android-build-box \
  jarsigner -verbose -sigalg SHA256withRSA -digestalg SHA-256 \
  -keystore signing.keystore \
  app-release.aab \
  playhex-alcalyn-key

# check it is signed: should list META-INF/PLAYHEX-.RSA
unzip -l app-release.aab | grep -E "META-INF/.*\.RSA"
```

Gradle runs as root in the container, so `app/build/` ends up owned by root.
If you also build outside docker: `sudo chown -R $USER app/build build`.

### 3. Upload on Google Play

/!\ **Warning**: firefox always get an error when uploading, works only on chrome (not chromium)...

`app-release.aab` should not be committed, check `.gitignore`.

- go to https://play.google.com/console, app "PlayHex" (`org.playhex.twa`)
- "Test and release" > "Production" (or "Closed testing" first) > "Create new release"
- upload `app-release.aab` (at the root of the project)
- release name: `1.0.0.10 (2026091900)`
- release notes: copy `fastlane/metadata/android/<lang>/changelogs/<versionCode>.txt`,
  one language at a time (they are not uploaded automatically, there is no fastlane `Fastfile` here)
- "Save" > "Review release" > "Start rollout to Production"

## Sign an apk

Sign apk with my signature.

Given `.apk` and `signing.keystore` are in same folder:

``` bash
docker run -it --rm -v `pwd`:/project mingc/android-build-box bash

# sign
/opt/android-sdk/build-tools/36.0.0/apksigner sign \
  --ks signing.keystore \
  org.playhex.twa_2025090823.apk

# verify
/opt/android-sdk/build-tools/36.0.0/apksigner verify \
  --verbose --print-certs-pem --min-sdk-version 24 \
  org.playhex.twa_2025090823.apk
```

## Rebuild TWA android app

Generates apk and source code.
Source code generated from pwabuilder should be compared to this repo.

Requires my signing key.

- go to https://www.pwabuilder.com/reportcard?site=https://playhex.org
- click "Package for stores"
- click "All settings"
- Fill form:
    - Version: create next "1.0.0.X"
    - Version code: set date+hour "2025090212" for 2 sept 2025, 12h
    - Include source code: check
    - Splash fade out duration: set "150"
    - Display mode: Standalone
    - Signing key: "Use mine", should open new inputs:
        - from my vault, "playhex key"
        - send file signing.keystore
        - fill rest from info in signing-key-info.txt
    - Submit

Then check for diff between pwabuilder download and this repo:

``` bash
diff -wqra . ~/Downloads/PlayHex
```

`PlayHex.aab` file can be uploaded to google console.

## F-Droid

Doc: https://f-droid.org/fr/docs/Submitting_to_F-Droid_Quick_Start_Guide/

To update on F-Droid:

- Update version in `app/build.gradle`:

```
        versionCode 2025090412
        versionName "1.0.0.3"
```

- commit and push
- Fork and clone `git@gitlab.com:fdroid/fdroiddata.git`
- Fork and clone `git@gitlab.com:fdroid/fdroidserver.git`

Folder structure must be like:

```
|- (current) playhex-android-app/
|- fdroiddata/
|- fdroidserver/
```

- Add in `../fdroiddata/metadata/org.playhex.twa.yml`:

``` yml
  - versionName: 1.0.0.3      # update, same as app/build.gradle versionName
    versionCode: 2025090412   # update, same as app/build.gradle versionCode
    commit: '2025090412'      # update, same as app/build.gradle versionCode
    subdir: app
    gradle:
      - yes
```

- Edit in `../fdroiddata/config.yml`:

``` yaml
serverwebroot: /tmp
```

- Run:

``` bash
docker run --rm -itu vagrant --entrypoint /bin/bash \
    -v $(pwd)/../fdroiddata:/build:z \
    -v $(pwd)/../fdroidserver:/home/vagrant/fdroidserver:Z \
    registry.gitlab.com/fdroid/fdroidserver:buildserver

. /etc/profile
export PATH="$fdroidserver:$PATH" PYTHONPATH="$fdroidserver"
export JAVA_HOME=$(java -XshowSettings:properties -version 2>&1 > /dev/null | grep 'java.home' | awk -F'=' '{print $2}' | tr -d ' ')
cd /build

mkdir -p /tmp/repo/status
sudo chmod 600 config.yml

fdroid readmeta
fdroid rewritemeta org.playhex.twa
fdroid checkupdates --allow-dirty org.playhex.twa
fdroid lint org.playhex.twa
fdroid build org.playhex.twa
```
