---
layout: post
category : dotnet
author: Chase Jenkins & Lucas Natraj
tags: [dnx, code]
title: dnxmon — Monitor for Changes in a DNX Application
---

dnxmon is a simple function that internally uses [nodemon](https://www.npmjs.com/package/nodemon) to watch folders and restart a **dnx (dotnet)**  application

### Requirements
- [node.js](https://nodejs.org/en/)
- [nodemon](https://www.npmjs.com/package/nodemon) node module (installed globally)
    ```bash
    npm install -g nodemon
    ```

### Install dnxmon
Add the following to your ~/.bash_profile

```bash
function dnxmon() {
    # Run dnx server continuously with nodemon
    # watching for changes to cs or json files
    # Usage:
    #   dnxmon <directory> <command>
    # dnxmon (applies the defaults: current directory and the "web" command)

    function dnxmonFn() {
        nodemon --ext "cs,json" --exec "dnx $1"
    }

    if [[ $# -eq 0 ]]
    then
        echo "running default ..."
        echo "nodemon --ext "cs,json" --exec "dnx web""
        dnxmonFn web
    else
        if [[ $# -eq 1 ]]
        then
            echo "nodemon --ext "cs,json" --exec "dnx $1""
            dnxmonFn $1
        else
            printf "syntax:\n"
            printf "  dnxmon             - will run dnxmon with 'web' as the default command\n"
            printf "  dnxmon <command>   - will run dnxmon with the given command\n"
        fi
    fi
}
```

### Running
dnx applications can now be launched with dnxmon rather than dnx.
```bash
$ dnxmon web
```
