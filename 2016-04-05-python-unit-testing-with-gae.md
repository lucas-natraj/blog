---
layout: post
category : web
author: Lucas Natraj, Torbjørn Vik & Chase Jenkins
tags: [quick, tutorial]
title: Python Unit Testing with Google App Engine
---

## Introduction
The preferred way of working in Python is to use [Virtual Environments](http://docs.python-guide.org/en/latest/dev/virtualenvs/), which provide an isolated environment for each project. This prevents projects and their dependencies from conflicting with each other.


## Site-specific configuration hook ([link](https://docs.python.org/2/library/site.html))
A `sitecustomize.py` file on the PYTHONPATH (environment variable) will automatically be imported by `site.py` and any lines added to that file are executed when Python starts.

PyCharm will add the source / content folders onto the PYTHONPATH, therefore providing an easy way to insert 'setup' logic before any subsequent execution occurs.
This capability will be leveraged to run any initialization logic that needs to be executed before running anything within the project (at least when running within PyCharm).


## Basic Project Setup

```text
.   <---------------------------- [project folder]
├── package1
|   └── some_code.py
|   └── ...
├── tests
|   ├── test_some_code.py
|   └── ...
├── app.yaml    <---------------- [GCP application configuration file]
├── appengine_config.py    <----- [automatically loaded when Google App Engine starts]
├── circle.yml      <------------ [CircleCI configuration file]
├── requirements.txt    <-------- [specifies project dependencies]
├── sitecustomize.py    <-------- [hook used when running within PyCharm]
└── test_runner.py      <-------- [runs all tests. used by 'continuous delivery']
```


## Creating a Virtual Environment with [virtualenv](https://pypi.python.org/pypi/virtualenv/)

From a terminal window at the project folder -

```bash
# virtualenv <path-to-virtual-environment-folder> 
$ virtualenv ~/work/envs/proj
```

Common practice is to keep the virtual env (venv) folder outside of the project structure.

After the venv has been created, it must be activated.

```bash
# source <path-to-venv-folder>/bin/activate
$ source ~/work/envs/proj/bin/activate

(proj) $        <----- prompt should now be updated to indicate virtual environment
```


## Installing project dependencies

From the terminal window at the project folder -

```bash
$ pip install -r requirements.txt
```

This will install all the packages specified in the `requirements.txt` file into the virtual environment.


## Launch PyCharm

It is best to launch PyCharm from within the activated Virtual Environment via

```bash
(proj) $ charm .
```

This ensures that various environment variables are available to the application.

## Contents of requirements.txt

```
# This requirements file lists all third-party dependencies for this project.
# Please see README.md for instructions on how to install the dependencies.

Flask==0.10
Flask-Cors==1.10.3
google-api-python-client
appengine-sdk==1.9.33.post0
python-dateutil
httplib2
pyjwt==1.4.0
oauth2client
mock==1.3.0
jsonschema
geopy
bunch

# Required for tests
Flask-Testing
pytest

# Required for code coverage
coverage
coveralls

# Required for unittest xml report creation
unittest-xml-reporting

# The AppEngine stubs used for unit tests requires PIL
Pillow==2.2.1

```

## Structure of a test file (./tests/test_some_code.py)

```python
import unittest

from google.appengine.api import search
from google.appengine.ext import ndb
from google.appengine.ext import testbed

from package1 import some_code

class SomeCodeTests(unittest.TestCase):
    def setUp(self):
        # First, create an instance of the Testbed class.
        self.testbed = testbed.Testbed()

        # Then activate the testbed, which prepares the service stubs for use.
        self.testbed.activate()

        # Next, declare which service stubs you want to use.
        self.testbed.init_all_stubs()

    def tearDown(self):
        self.testbed.deactivate()

    def test_something(self):
        result = some_code.return_true()
        self.assertTrue(result)
        
    # ...
```

## Content of test_runner.py

```python
import sys
import unittest

import os
import xmlrunner


class Context:
    def __init__(self):
        pass

    @staticmethod
    def setup_gae_paths():
        import dev_appserver

        print 'setting up google app engine environment ...'
        dev_appserver.fix_sys_path()

    @staticmethod
    def import_appengine_config():
        # Loading appengine_config from the current project ensures that any
        # changes to configuration there are available to all tests (e.g.
        # sys.path modifications, namespaces, etc.)
        print 'importing appengine_config ...'
        try:
            import appengine_config
        except ImportError:
            print "Note: unable to import appengine_config."


def run_tests(gen_xml=True, test_path='.'):
    suite = unittest.TestLoader().discover(test_path)
    if gen_xml:
        result = xmlrunner.XMLTestRunner().run(suite)
    else:
        result = suite.run(unittest.TestResult())
        print 'summary: ' + str(result)
        if len(result.failures) > 0:
            print 'failures:'
            for failure in result.failures:
                print '   ' + str(failure)
        else:
            print 'success!'

    # Return an exit code indicating the number of failures
    sys.exit(len(result.failures))


def main(argv=None):
    import getopt

    # check for Virtual Env
    if os.environ.get('VIRTUAL_ENV') is None:
        raise Exception("Must be run within a virtual environment")

    # initialize variables
    gen_xml = True
    test_path = "."

    # process options
    opts, args = getopt.getopt(sys.argv[1:], 'x:p:', ['xml=', 'path='])
    for o, a in opts:
        if o in ("-x", "--xml"):
            gen_xml = a == 'True'
        elif o in ("-p", "--path"):
            test_path = a

    # setup environment
    Context.setup_gae_paths()
    Context.import_appengine_config()

    # go!
    run_tests(gen_xml, test_path)


if __name__ == '__main__':
    sys.exit(main())


```

## Content of sitecustomize.py

This file is automatically invoked by PyCharm so it can be used to setup the desired paths to the required google packages.

```python
from test_runner import Context

# auto-execute when running within PyCharm (or any environment that honors sitecustomize.py)
Context.setup_gae_paths()

```


## Content of appengine_config.py

```python
# coding=utf-8
"""
`appengine_config.py` is automatically loaded when Google App Engine starts a new instance of your application.
This runs before any WSGI applications specified in app.yaml are loaded.
This file ensures that the python lib directory is added to the path. Without it, the flask module will not be found.
"""

import os

from google.appengine.ext import vendor

# Add the correct packages path if running on Production (this is set by Google when running on GAE in the cloud)
if os.getenv('SERVER_SOFTWARE', '').startswith('Google App Engine/'):
    vendor.add('venv/lib/python2.7/site-packages')

```


## Contents of circle.yml

```yaml
dependencies:
  pre:
    # Upgrade pip
    - pip install --upgrade pip

machine:
  python:
    version: 2.7.5

test:
  override:
   - coverage run test_runner.py
  post:
   - mkdir -p $CIRCLE_TEST_REPORTS/junit/
   - cp ./TEST*.xml $CIRCLE_TEST_REPORTS/junit/
   - coveralls

deployment:
  prod:
    branch: master
    commands:
    
     # Ensure that 3rd party packages are bundled
     - ln -s $VIRTUAL_ENV ./venv
     
     # Install gcloud
     - curl -sSL https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-sdk-97.0.0-linux-x86_64.tar.gz | sudo tar zxfv - -C $HOME
     
     # Deploy using gcloud
     - echo $GCLOUD_DEPLOY_KEY > key.json
     - sudo $HOME/google-cloud-sdk/bin/gcloud auth activate-service-account circleci-deploy@googproject.iam.gserviceaccount.com --key-file key.json
     - sudo $HOME/google-cloud-sdk/bin/gcloud -q preview app deploy --project googproject --version 1 app.yaml

```


## PyCharm Setup

The only thing that is absolutely necessary to set up is the location of the Python Intepreter.

`Preferences ->  Project -> Project Interpreter : <path-to-virtual-env>`

![Example](./assets/2016-04-05-pycharm-testing-GAE-interpreter-config.png)


## Execution within PyCharm

The following images illustrate the various configurations for debugging / running within PyCharm


### Unittest Configuration

![Example](./assets/2016-04-05-pycharm-testing-GAE-unittests-config.png)


### TestRunner Configuration

![Example](./assets/2016-04-05-pycharm-testing-GAE-testrunner-config.png)


### Development Application Server Configuration

![Example](./assets/2016-04-05-pycharm-testing-GAE-devappserver-config.png)


### Running Tests from the Shell

Run from the project folder
```bash
$ python -m unittest discover --pattern=test_*.py
```
