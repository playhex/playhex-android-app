To check build before submit to fdroid:

Update version in `app/build.gradle`:

```
        versionCode 2025090412
        versionName "1.0.0.3"
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
