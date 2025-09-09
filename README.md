# Android app

App id: `org.playhex.twa`

This repo contains source code for [PlayHex](https://playhex.org) TWA.
It is needed at least for F-Droid to build apk and add it in its store.

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
    - Splash fade out duration: set "50"
    - Display mode: Standalone
    - Signing key: "Use mine", should open new inputs:
        - from my vault, "playhex key"
        - send file signing.keystore
        - fill rest from info in signing-key-info.txt
    - Submit

Then check for diff between pwabuilder download and this repo:

```
diff -wqra . ~/Downloads/PlayHex
```

`PlayHex.aab` file can be uploaded to google console.

## F-Droid

Doc: https://f-droid.org/fr/docs/Submitting_to_F-Droid_Quick_Start_Guide/

To check build before submit to fdroid:

Update version in `app/build.gradle`:

```
        versionCode 2025090412
        versionName "1.0.0.3"
```

Update version AND commit in fdroiddata:

Fork and clone `git@gitlab.com:fdroid/fdroiddata.git`

Add in `metadata/org.playhex.twa.yml`:

``` yml
  - versionName: 1.0.0.3
    versionCode: 2025090412
    commit: 47e04282d53eaa340c03a27b88331a8ad09d7a61
    subdir: app
    gradle:
      - yes
```

Run:

``` bash
docker run --rm -itu vagrant --entrypoint /bin/bash \
    -v /..pwd.../dev/fdroiddata:/build:z \
    -v /..pwd.../dev/fdroidserver:/home/vagrant/fdroidserver:Z \
    registry.gitlab.com/fdroid/fdroidserver:buildserver

. /etc/profile
export PATH="$fdroidserver:$PATH" PYTHONPATH="$fdroidserver"
export JAVA_HOME=$(java -XshowSettings:properties -version 2>&1 > /dev/null | grep 'java.home' | awk -F'=' '{print $2}' | tr -d ' ')
cd /build

mkdir -p /tmp/repo/status

# in config.yml, set:
# serverwebroot: /tmp

fdroid readmeta
fdroid rewritemeta org.playhex.twa
fdroid checkupdates --allow-dirty org.playhex.twa
fdroid lint org.playhex.twa
fdroid build org.playhex.twa
```
