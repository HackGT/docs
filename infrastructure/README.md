# Motivation

HackGT has a lot of apps it needs to run for a **long time**.

I know, long running apps are scary. We see this in past HackGT apps that are magically fixed with a restart.
Some of the apps we run are also kinda crap, and we would like one person's crappy app to not interfere
with everyone else's apps.
As a team we also have a lot of opinions, some like `js`, some `python`, some `haskell`, etc. so we want to include
everyone (`<serious>` because everyone is important at HackGT `</serious>`).
Finally we want our system to be clear, and easily understood by future members, containing the least possible
"magic setup" as possible.

We would like to automate all of this, so we want a server management tool that will get us:

# Goals & Non-Goals

 - Be Language-Agnostic (does not work "better" with any particular platform)
 - Be Portable (minimal platform dependency)
 - Be Clear (all server management actions are a directly infered from the source)
 - Be Declarative (can rollback source to rollback server)
 - Be Siloed (all dependencies and resources of an app are separated from others)
 - Be Easy (if we want this to stick, its got to be easy to use)

# Design

## Contain Yourself

`Docker` is an easy solution to the silo and portable bullets (although its not very declarative or clear),
so we will use it to contain every app that we want to run.
It doesn't matter that `Docker` isn't declarative since we don't care if the actual apps are.

I know, maybe some of you wanted `lxce` or `nixOS`, I'm sorry.

## Freight

We will be using Kubernetes, since its gives us an easy way to run a an arbitrary amount of containers
at once.

## The App Repo

Often a design can be gleaned from thinking of "what we would like to happen".
So if I'm writing an app for a hackathon I would like it to be portable across all events,
and in doing so I do not want to be responsible for placing this app in any specific event.

So the steps to writing an app would be:

1. Write your damn app
2. Create a [Dockerfile](https://docs.docker.com/engine/reference/builder/) that runs your app.
3. Create a `deployment.yaml` file which specifies what resources your app needs and
   what other containers (if any) it depends on. This will be equivalent to the `spec.containers`
   section of a [K8s deployment configuration](http://kubernetes.io/docs/user-guide/deployments/).

Now when building or running the docker image the developer will have access to the entire event metadata
(described later) in the form of JSON in an environment variable:

```bash
HACKATHON_METADATA='{"name":"HackGT","domain":"hack.gt", ...}'
```

This is helpful if you want to display the name of the event, etc. in your app.

### Tests

Every file that matches `/tests/Dockerfile` will spawn and run during testing.
If you test different sections of your app, say one is called `websocket`, you can test
the websocket capabilities of your app by writing tests in `websocket/tests/` then making
a container to run the tests in.

## The Event Repo

When we know what event we are running these apps on we get to know a lot more,
this is a simple repo that contains metadata about each event:

```yaml
name: "HackGT"
times:
  - start: 12345
  - end: 12345
  - submission: 12345
homepage: "https://hack.gt/"
domain: "hack.gt"
apps:
  hackgt/helpq:
    - branch: master # this is default
    - url: helpmewith.hack.gt
  hackgt/eventbriter:
    - domain: tickets
  etc..
```

This metadata gets its own repository and is given a special name: `hackathon.yaml`.

## The CI

So far so simple I think. We've pushed off most of the responsibilities away from the people writing
the apps: they only need to provide a container and what resources they want to use (ports, DBs, etc.).

The CI therefore, by process of elimination, will do all the work for us. Specifically it will:

 - Spawn an instance of any app that is under review through a pull request
 - Run tests on all apps under review through a pull request
 - Find all `hackathon.yaml` files and
   - Create a namespace for that event
   - Create a load balancer for that hackathon behind its `metadata.domain`
   - Run all apps' tests
   - Spawn instances of all apps related to that hackathon
   - Create a `hackathon.lock` file on each successful instantiation of all apps.

This makes it easy to make a 'development event' in which all the development builds of all the apps
are tested, kind of like having a conventional `dev` branch on each repo.
This also makes sure that the development server will be held up to the same standards as all other events.

Ideally there should be no reason to edit anything in the CI server, since all it does is scan repos
for pull requests and `hackathon.yaml` files. So what CI shoud we use?

### What's in a CI?

My understanding of the CI landscape leads me away from conventional task runners and more towards CI that's geared
towards pipelines. Pipelines are a very clear and intuitive flow of jobs.
I would also like it to be difficult to make a change in the CI process that is not mirrored in git, since
it will be running on k8s, it should be able to destroyed and rebuilt with ease.

I prefer to not use Jenkins, as it does not support pipelines as first-class entities and makes it too easy
to specify a script to be run in the build process that never sees the light of version control.

I also prefer to not use any cloud services since it would make the entire system less portable.

For these reasons I am partial to [Concourse](https://concourse.ci/index.html), they have some nice
[propoganda](https://concourse.ci/concourse-vs.html) that will help the koolaid go down.
