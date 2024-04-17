# Divergent `proto` builds

This is just to validate that `python` is fine with importing generated gRPC
clients from divergent `proto` directories with overlapping package naming.

- I used `python3` here.
- I haven't used `python` in any form, for many many many years, so forgive me
  for having lost all my pythonic manners.

## Setup

Validate that clients can be built for 2 separate and independent `proto`
directories, with overlapping package names. In this case:

- `proto-a`
- `proto-b`

```shell
# ensure we're tidy.
for X in a b; do
    Y="proto-${X}"
    buf lint ${Y}
    buf format -w ${Y}
    tree ${Y}
done;
```

## Build Clients

I haven't written python in a long time, so I'm following
[the `venv` instructions here](https://grpc.io/docs/languages/python/quickstart/#prerequisites) mostly.

```shell
virtualenv venv
source venv/bin/activate
python -m pip install --upgrade pip
python -m pip install grpcio
python -m pip install grpcio-tools
python -m pip install ipython

for X in a b; do
    Y="proto-${X}"
    Z="${X}${X}${X}${X}"
    python3 -m grpc_tools.protoc -I${Y} --python_out=. --pyi_out=. --grpc_python_out=. ${Y}/github/canardleteer/${Z}/v1alpha1/service.proto
done;
tree github
```

## Use Clients

I used `ipython` to explore.

```shell
ipython
```

In another terminal, you can spin up a faux server:

```shell
nc -l localhost 50051
```

```python
import grpc
import importlib.util

# It's been many many years since I've used python, so I assume this could be
# drastically reduced.
import github.canardleteer.aaaa.v1alpha1.service_pb2 as aaaa
spec=importlib.util.spec_from_file_location("github.canardleteer.aaaa.v1alpha1.service_pb2_grpc", "github/canardleteer/aaaa/v1alpha1/service_pb2_grpc.py")
aaaa_grpc = importlib.util.module_from_spec(spec)
spec.loader.exec_module(aaaa_grpc)

import github.canardleteer.bbbb.v1alpha1.service_pb2 as bbbb
spec=importlib.util.spec_from_file_location("github.canardleteer.bbbb.v1alpha1.service_pb2_grpc", "github/canardleteer/bbbb/v1alpha1/service_pb2_grpc.py")
bbbb_grpc = importlib.util.module_from_spec(spec)
spec.loader.exec_module(bbbb_grpc)

# Test Service A.
with grpc.insecure_channel('localhost:50051') as channel:
    stub = aaaa_grpc.SomeATypeServiceStub(channel)
    response = stub.SomethingUnimplemented(aaaa.SomethingUnimplementedRequest())

    # Confirm you see something in your `nc` window.

    # For me, it looks like:
    # -> nc -l localhost 50051
    # PRI * HTTP/2.0
    # 
    # SM
    # 
    # $���@@@?

    # Hit Ctrl-C in ipython, the request will never be responded to.

# Kill your nc process, and relaunch it the same way as before: nc -l localhost 50051

# Test Service B.
with grpc.insecure_channel('localhost:50051') as channel:
    stub = bbbb_grpc.SomeBTypeServiceStub(channel)
    response = stub.SomethingUnimplemented(bbbb.SomethingUnimplementedRequest())

    # Confirm you see something in your `nc` window.

    # For me, it looks like:
    # -> nc -l localhost 50051
    # PRI * HTTP/2.0
    # 
    # SM
    # 
    # $���@@@?

    # Hit Ctrl-C in ipython, the request will never be responded to.
```

## Shrinking

- I assume you can move all the `importlib` stuff into something much more
  convenient.
- I assume you can also move generated python files after a build, into a
  flatter directory, if that's wanted.
