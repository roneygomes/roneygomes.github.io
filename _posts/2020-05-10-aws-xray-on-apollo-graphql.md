---
layout: post
date: 2020-05-10
title: AWS X-Ray on Apollo GraphQL
tags: [aws, x-ray, graphql, apollo]
---
After getting abducted by a soul-sucking vortex of mentally debugging AWS code,
hand-waving documentation, Stack Overflow posts and github issues I finally
managed to get X-Ray working on an Apollo GraphQL API at work. This is me
explaining how I got there in the hopes that other people won't have to spend
as much time as I did. It's also an attempt to consolidate all this new
knowledge I got so that I don't forget it by next month or so. I am not sure I
will have to ever do this again but who knows? I was looking for a reason to
write something technical anyway.

This post assumes you already have the X-Ray agent running somewhere accessible
by your GraphQL API. It can be a process running on the same machine as your
application, a kubernetes daemonset exposed via a service, it doesn't matter.
We'll assume your application can talk to it.

## Tracing Inbound Requests

X-Ray has two modes of operation: **auto** and **manual**. What does that
actually mean? It means that you either manage the life cycle of trace segments
yourself or you can let the SDK do most of the work and jump aboard only when
necessary. We really want to live in auto mode here as this provides a handful
of very cool things like free traces for outbound requests to other AWS
services like RDS, S3 and DynamoDB.

So, how to intercept incoming requests? Before we start, it's important to make
it very clear that this post **only applies** to applications being served by
Express. Similar steps may work for a different server like Hapi, but since
Express is the only framework officially supported by the X-Ray SDK, other
frameworks will take more effort since they'll have to operate on manual mode.

### Dependencies

Make sure to get the latest version of the SDK:

```bash
npm i aws-xray-sdk
```

### Configuration

Set the required configs on server startup.

```tsx
import AWSXRay from "aws-xray-sdk";

AWSXRay.setDaemonAddress("<x-ray-host>:<x-ray-port>");
```

If your application has queries that can possibly return many individual itens
in the response object, consider also setting X-Ray's streaming threshold.

```tsx
AWSXRay.setStreamingThreshold(0);
```

By setting the streaming threshold to zero, we are forcing the X-Ray client to
send each trace subsegment individually to the X-Ray agent. The standard
behaviour is to group subsegments in batches but since — depending on the query
— we can trigger tens of resolvers in a single request, it's not that hard to
have batches which size exceeds the maximum allowed. UDP is the only language
X-Ray speaks so we're bound to a ceiling of 65 KB per packet.

Feel free to adjust that value to something that better fits your application's
profile.

X-Ray has a bunch of other settings that can also be useful to your particular
use case so don't forget to check the docs[^1].

### Enabling X-Ray's Middleware

Get ahold of the Express instance and apply X-Ray's middleware before and after
the Apollo server is created. It should look more or less like the following:

```tsx
import express from "express";
import { ApolloServer } from "apollo-server-express";

const app = express();
app.use(AWSXRay.express.openSegment("application-name-here"));

const server = new ApolloServer({
    // schema, data sources etc.
});

server.applyMiddleware({ app, path: "/"});
app.use(AWSXRay.express.closeSegment());
```

There we go. Now incoming requests are going to have a corresponding trace
showing up on X-Ray's console in AWS with our application being displayed on
the service map as well. Since we're on auto mode, traces will also respect any
sampling rule we might have previously set.

### Tracing Resolvers

Having tracing enabled at the root endpoint is not enough, we also need to
enrich our traces with the execution times of each resolver triggered during a
request. For that to be possible we will need to leverage Apollo's extension
API so that we can hook into the resolvers' life cycle and set time stamps for
their corresponding start and end times.

This is how we can create a custom X-Ray tracing extension.

```tsx
class XRayTracing implements GraphQLExtension<AppContext> {
  public willResolveField(source, args, context, info) {
    const subsegment = AWSXRay.getNamespace().runAndReturn(() => {
      const segment = AWSXRay.getSegment();

      if (!segment) {
        return null;
      }

      return segment.addNewSubsegment(info.fieldName);
    });

    return () => {
      if (subsegment) {
        subsegment.close();
      }
    };
  }
}
```

The `willResolveField` method is called by Apollo whenever a node in the graph
is about to be resolved. There's no better place to start tracking a resolver's
execution time.

The first thing we need to do is to grab the `Segment`[^2] instance added by
X-Ray's middleware when the request started. Once we have the segment we can
add a `SubSegment`[^3] to it corresponding to our resolver's portion in the
trace.

By the way, if the terms segment and subsegment are new to you, don't worry.
They are quite simple in practice.

Think of a `Segment` as a slice of time. We can `open()` and `close()` them.
The time it takes between these two method calls is the segment's interval,
length or time window. A `Segment` can have many `Subsegment`s, which are also
slices of time that can be closed and opened. The difference here is that
subsegments are always part of a segment. They serve as way to add more detail
to traces.

Internally X-Ray uses something called continuation local storage to store the
current segment's instance. Similar to a thread-local storage mechanism.

[^1]: [X-Ray Node Docs]()
[^2]: [Segment]()
[^3]: [SubSegment]()
