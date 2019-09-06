---
layout: post
category: google-cloud
author: Chase Jenkins & Lucas Natraj
---

## Intro

We've encountered a number of ways to write API's on Google App Engine (GAE) using Python, each with their advantages and drawbacks. Here we'll discuss some of them.

### Basic Setup for all Frameworks

Getting started with GAE using Python is quick. At a bare minimum, you need your `app.yaml` file and a primary application file — let's call it `main.py`.

Here's a simple `app.yaml` file. This file is used to configure the setup for your application.

```yaml
application: my-google-project-id
version: 1
runtime: python27
api_version: 1
threadsafe: yes

handlers:
- url: /.*
  script: main.app
```

The `application` parameter should match your google project id. So, if your google project id is "my-test-app", put "my-test-app" there. The `version` parameter is also fairly straightforward. Since we're using Python 2.7 (as of this writing, the only supported version), use "python27" as the `runtime` value. The `threadsafe` parameter tells GAE whether or not a single instance can handle multiple simultaneous requests. If it can, it should be set to "yes" or "true".

The `handlers` section is where the top-level routing is handled. You can have different handler objects for different sub-paths, if you like. Here, we're using `/.*` to tell GAE to direct all requests to `main.app`, which is where our actual code will live.

## Flask

Now that we have an `app.yaml` created, let's put some code in `main.py`.

```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/items', methods=['GET'])
def get_all_items():
    return jsonify({'items': [{'id': 1, 'name': 'cereal'}, {'id': 2, 'name': 'banana'}]})
```

This is clearly a very simple endpoint, but a few things are happening here worth mentioning. First, we import a couple of things from the flask library — `Flask` for creating the `app` (which is what is referenced in `app.yaml`), and `jsonify` for easily converting python dictionaries to JSON. The `get_all_items()` method contains the code that we want to execute for a given endpoint, and the `@app.route` decorator specifies the url path that will invoke that function. Note that the function can be named anything — it's the `@app.route` decorator that specifies that path where teh endpoint will be accessible.

One other important detail here — we need to install the flask library, since we have a dependency on it. We need to add two more files to make this possible. The first is a `requirements` file which lists our dependencies:

```python
Flask==0.10
```

To install the dependencies in this file to the `lib` subfolder in your project, run `pip install -r requirements -t lib`.

Finally, we need to make sure the contents of `lib` are loaded into our application. Create a file called `appengine_config.py` with these contents:

```python
from google.appengine.ext import vendor

vendor.add('lib')
```

