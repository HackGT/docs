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

Now when building or running the docker image the developer will have access to the following environment variables:
 - `HGT_EVENT_NAME="HackGT"`
 - `HGT_EVENT_START="YYYY/MM/DD HH:MM:SS`
 - `HGT_EVENT_END="YYYY/MM/DD HH:MM:SS`
 - `HGT_HOME_URL="https://hack.gt/"`
 - `HGT_HELPQ_URL="https://helpmewith.hack.gt/"` (might be empty if no help queue)

## The Event Repo

When we know what event we are running these apps on we get to know a lot more,
this is a simple repo that contains metadata about each event:

```yaml
name: "HackGT"
times:
  - start: 12345
  - end: 12345
  - submission: 12345
homepage: ""
domain: ""
apps:
 - "repo 1"
 - "repo 2"
 - etc.
```
