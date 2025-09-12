# Android app

App id: `org.playhex.twa`

This repo contains source code for [PlayHex](https://playhex.org) TWA.
It is needed at least for F-Droid to build apk and add it in its store.

## Build locally

https://hub.docker.com/r/mingc/android-build-box/

``` bash
docker run -it --rm -v `pwd`:/project mingc/android-build-box bash

# build aab
./gradlew bundle
```

### Sign

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
  - versionName: 1.0.0.3      # update
    versionCode: 2025090412   # update
    commit: 47e04282d53eaa340c03a27b88331a8ad09d7a61  # update
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
