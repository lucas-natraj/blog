---
layout: post
category: google-cloud
author: Chase Jenkins & Lucas Natraj
---

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

```
Flask==0.10
```

To install the dependencies in this file to the `lib` subfolder in your project, run `pip install -r requirements -t lib`.

Finally, we need to make sure the contents of `lib` are loaded into our application. Create a file called `appengine_config.py` with these contents:

```python
from google.appengine.ext import vendor

vendor.add('lib')
```

To run your app locally, run `dev_appserver.py .` from the command line.

### Authentication with OAuth2

Now that we have a basic flask app with an endpoint, let's add OAuth2 authentication. Depending on what you want to do with the authentication, there are a few options.

First, if you simply want to make sure the user is authenticated and get basic user information, you can use the `google.appengine.api` library:

```python
import google.appengine.api
from flask import Flask, jsonify

app = Flask(__name__)
required_oauth_scope = 'https://www.googleapis.com/auth/userinfo.email'

@app.route('/items', methods=['GET'])
def get_all_items():
    user = get_current_user(required_oauth_scope)
    if user is None:
        return jsonify({'error': 'not authenticated'}), 403
    return jsonify({'items': [{'id': 1, 'name': 'cereal'}, {'id': 2, 'name': 'banana'}]})
```

This code attempts to get the current user who has authenticated with the specified OAuth2 *scope*. If there is no authenticated user, it returns an error message with a 403 (Forbidden) status code. More details on this, including checking OAuth2 client IDs, can be found [here](https://cloud.google.com/appengine/docs/python/oauth/).

### More Complex OAuth2 Workflows

What happens if we need to use the user's credentials to call other APIs on their behalf? For example, what if our API needs to call the Google Cloud Storage API to list files in a bucket owned by that user? Now we need to extract the OAuth2 access token from either the headers or the query string.

```python
from flask import request

token = None

if 'access_token' in request.args:
    token = request.args.get('access_token')
elif 'Authorization' in request.headers:
    auth_header = request.headers.get('Authorization')
    split_header = auth_header.split()
    if split_header[0] == 'Bearer':
        token = split_header[1]
```

Here, we import the `request` object from flask, which allows us to access the query string arguments (`request.args`) as well as the headers (`request.headers`). OAuth2 access tokens are passed as either *Bearer* tokens in an *Authorization* header, or in the *access_token* query string argument.

With this token, additional google APIs can now be called on behalf of the user.

*Coming soon...*

## Cloud Endpoints

While the Flask framework allows us to get up and running quickly on an evolving API, *Google Cloud Endpoints* gives us a much more structured API implementation. A benefit of this structure is the ability to use the Google API Explorer to interact with our API, as well as built-in support for enforcing authentication.

First, let's create our app.yaml with endpoints support.

```yaml
application: my-google-endpoints-project-id
version: 1
runtime: python27
api_version: 1
threadsafe: yes

handlers:
- url: /_ah/spi/.*
  script: services.application

libraries:
- name: endpoints
  version: 1.0
- name: pycrypto
  version: latest
```

Here, we add the endpoints and pycrypto libraries that are provided by google (no *pip install* needed). Additionally, we create the handler for our application. Note that that the url of `/_ah/spi/.*` cannot be changed and is required for API Explorer support.

Next, we'll create `services.py` where we expose our api object.

```python
import endpoints
import api

application = endpoints.api_server([api.MyApi])
```

Here, we import the *endpoints* library as well as our *api* module, which we'll create momentarily. We then initialize the `application` variable using the class we create in `api.py`, which we'll do now:

```python
import endpoints
from protorpc import messages
from protorpc import message_types
from protorpc import remote

class Item(messages.Message):
    id = messages.IntegerField(1)
    name = messages.StringField(2)


class GetAllItemsResponse(messages.Message):
    items = messages.MessageField(Item, 1, repeated=True)


@endpoints.api(name='items', version='v1', description='Items API')
class ItemsApi(remote.Service):

    @endpoints.method(message_types.VoidMessage, GetAllItemsResponse, name='items', path='items', http_method='GET')
    def get_all_items(self, request):
        items = [];
        items.append(Item(id=1, name='cereal'))
        items.append(Item(id=2, name='banana'))

        return GetAllItemsResponse(items=items)
```


## No Framework

Coming soon...
