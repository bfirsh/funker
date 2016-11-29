# Funker

Functions as Docker containers.

Funker allows you to package up pieces of your application as Docker containers and have them run on-demand on a swarm.

You can define functions like this as Docker services:

```javascript
var funker = require('funker');

funker.handler(function(args, callback) {
  callback(args.x + args.y);
});
```

Then call them from other Docker services on any node in the swarm:

```python
>>> import funker
>>> funker.call("add", x=1, y=2)
3
```

These functions are being called demand, scale effortlessly, and make your application vastly simpler. It's a bit like serverless, but just using Docker.

## Getting started

### Creating a function

First, you need to package up a piece of your application as a function. Let's start with a trivial example: a function that adds two numbers together.

Save this code as `handler.js`:

```javascript
var funker = require('funker');

funker.handler(function(args, callback) {
  callback(args.x + args.y);
});
```

We also need to define the Node package in `package.json`:

```
{
  "name": "app",
  "version": "0.0.1",
  "scripts": {
    "start": "node handler.js"
  },
  "dependencies": {
    "funker": "^0.0.1"
  }
}
```

Then, we package it up inside a Docker container by creating `Dockerfile`:

```
FROM node:7-onbuild
```

And building it:

```
$ docker build -t add .
```

To run the function, you create a service:

```
$ docker network create --attachable -d overlay funker
$ docker service create --name add --network funker add
```

The function is now available at the name `add` to other things running inside the same network.

### Calling a function

Let's try calling the function from a Python shell:

```
$ docker run -it --net funker funker/python
```

(The `funker/python` image is just a Python image with the `funker` package installed.)

You should now see a Python prompt. Try importing the package and running the function we just created:

```python
>>> import funker
>>> funker.call("add", x=1, y=2)
3
```

Cool! So, to recap: we've put a function written in Node inside a container, then called it from Python. That function is run on-demand, and this is all being done with plain Docker services and no additional infrastructure.

## Implementations

There are implementations of handling and calling Funker functions in various languages:

- [Node](https://github.com/bfirsh/funker-node)
- [Python](https://github.com/bfirsh/funker-python)

## Deploying with Compose

Functions are just services, so they are really easy to deploy using Compose. You simply define them alongside your long-running services.

For example, to deploy a function called `process-upload`:

```yaml
version: "2"
services:
  web:
    image: oscorp/web
  db:
    image: postgres
  process-upload:
    image: oscorp/process-upload
    restart: always
```

In all the services in this application, the function will be available under the name `process-upload`. For example, you could call it with a bit of code like this:

```python
funker.call("process-upload", bucket="some-s3-bucket", filename="upload.jpg")
```

## Example applications

- [funker-example-voting-app](https://github.com/bfirsh/funker-example-voting-app) -- an example app that uses Funker to do processing in the background

## Architecture

The architecture is intentionally very simple. It leans on Docker services as the base infrastructure, and avoids any unnecessary complexity (daemons, queues, storage, consensus systems, and so on).

Functions run as Docker services. When they boot up, they open a TCP socket and sit there waiting for a connection.

To call functions, another Docker service connects to the function at its hostname. This can be done anywhere in a swarm due to Docker's overlay networking. It sends function arguments as JSON, then the function responds with a return value as JSON.

Once it has been called, the function refuses any other connections. Once it has responded, the function closes the socket and quits immediately. Docker's state reconciliation will then boot up a fresh copy of the function ready to receive calls again.

So, each function only processes a single request. To process functions in parallel, we need to have multiple warm functions running in parallel, which is easy to do with Docker's service replication. The idea is to do this automatically, but this is incomplete. [See this issue for more background and discussion.](https://github.com/bfirsh/funker/issues/4)

## Credits

- [Justin Cormack](https://github.com/justincormack) for the idea.
