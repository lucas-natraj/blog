---
layout: post
category : web
author: Lucas Natraj
tags: [Google, go, golang, appengine]
title: Google App Engine with Go
---

### Setup

1. Install [Go](https://golang.org)
2. Install the [go_appengine sdk](https://cloud.google.com/appengine/downloads)
3. Create a script with the following content:  

    ```bash
    # Update PATH for go_appengine
    export PATH=<path-to-go_appengine-sdk>:$PATH

    # Update GOPATH to current directory
    export GOPATH=`pwd`

    # Update PATH for the Google Cloud SDK.
    source '/Users/<user-name>/google-cloud-sdk/path.bash.inc'

    # Enable shell command completion for gcloud.
    source '/Users/<user-name>/google-cloud-sdk/completion.bash.inc'
    ```

4. '. source' the  script from the project root folder
    ```bash
    $ source path-to-script
    ```


### Project Structure

```text
.   <-------------------------------- [project folder]
├── bin
|   └── ...
├── pkg
|   ├── test_some_code.py
|   └── ...
├── scripts
|   └── cover.sh    <---------------- [script for generating test coverage]
├── src
|   ├── github.com
|   |   └── ...
|   ├── golang.org
|   |   └── ...
|   └── project
|       ├── app
|       |   ├── app.go
|       |   └── app.yaml    <-------- [GCP application configuration file]
|       ├── package1
|       |   ├── foo.go
|       |   └── foo_test.go
|       └── package2
|           ├── bar.go
|           └── bar_test.go
└── circle.yml      <---------------- [CircleCI configuration file]
```


### Useful Packages

* [`Gorilla Pat`](http://www.gorillatoolkit.org/pkg/pat) - Request router and dispatcher
* [`Gin Gonic`](https://gin-gonic.github.io/gin/) - Request routing middleware
* [`Testify`](testify golang assert) - Testing tools

### Processing Request Body Payloads


### Testing & Code Coverage

```bash
# coverage for a single package
go test -v -cover -coverprofile=profile.cov ./identity/user-service/... | /Users/lnatraj/work/data-lake/datalake-utils/identity/bin/go-junit-report > report.xml
```

#### Contents of cover.sh

```bash
#!/bin/sh
# Generate test coverage statistics for Go packages.
#
# Works around the fact that `go test -coverprofile` currently does not work
# with multiple packages, see https://code.google.com/p/go/issues/detail?id=6909
#
# Usage: cover path [--html | --coveralls]
#
#     path          Path to the root folder for test discovery
#     --html        Additionally create HTML report and open it in browser
#     --coveralls   Push coverage statistics to coveralls.io
#

set -e

work_dir=.cover
profile="$work_dir/cover.out"
html="$work_dir/cover.html"
mode=count

generate_coverage_data() {
    rm -rf "$work_dir"
    mkdir "$work_dir"

    for pkg in "$@"; do
        f="$work_dir/$(echo ${pkg} | tr / -).cover"
        go test -covermode="$mode" -coverprofile="$f" "$pkg"
    done

    # create merged coverage file
    echo "mode: $mode" >"$profile"
    grep -h -v "^mode:" "$work_dir"/*.cover >>"$profile"
}

show_cover_report() {
    go tool cover -${1}="$profile" -o ${html}
}

push_to_coveralls() {
    echo "Pushing coverage statistics to coveralls.io"
    goveralls -coverprofile="$profile"
}

generate_coverage_data $(go list ${1}/...)
show_cover_report func
case "$2" in
    "")
        ;;
    --html)
        show_cover_report html ;;
    --coveralls)
        push_to_coveralls ;;
    *)
        echo >&2 "error: invalid option: $2"; exit 1 ;;
esac
```


### Deploying

```bash
# deploy to google cloud platform
$ goapp deploy -application app_name -version 1 ./src/project/app/
```

#### Contents of circle.yml

```yaml
dependencies:
  pre:
    # go app engine
    - curl -o $HOME/go_appengine_sdk_linux_amd64-1.9.35.zip https://storage.googleapis.com/appengine-sdks/featured/go_appengine_sdk_linux_amd64-1.9.35.zip
    - unzip -q -d $HOME $HOME/go_appengine_sdk_linux_amd64-1.9.35.zip

    - sudo `which gcloud` components install --quiet app-engine-python
    - export GOPATH=$HOME/root/project; go get -t ./src/project/...

machine:
  python:
    version: 2.7.5

test:
  override:
   - export GOPATH=$HOME/root/project; go test -cover ./src/project/...
   - export GOPATH=$HOME/root/project; ./scripts/cover.sh ./src/project/ --html

  post:
   - cp ./project/.cover/cover.html $CIRCLE_ARTIFACTS/coverage.html

deployment:
  prod:
    branch: master
    commands:

      # Install gcloud
      - curl -sSL https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-sdk-97.0.0-linux-x86_64.tar.gz | sudo tar zxfv - -C $HOME
     
      # Deploy using gcloud
      - echo $GCLOUD_DEPLOY_KEY > key.json
      - sudo $HOME/google-cloud-sdk/bin/gcloud auth activate-service-account circleci-deploy@googproject.iam.gserviceaccount.com --key-file key.json
      - sudo export GOPATH=$HOME/root/project; $HOME/google-cloud-sdk/bin/gcloud -q preview app deploy --project googproject --version 1 ./src/project/app/app.yaml
```

### Links

* [App Engine with Go](https://cloud.google.com/appengine/docs/go/)
* [Uploading, Downloading, and Managing a Go App](https://cloud.google.com/appengine/docs/go/tools/uploadinganapp)
