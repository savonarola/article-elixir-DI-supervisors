Our task: 

We need siblings of supervisors to find out about each other.
Preferabbly we also want our supervisor tree to be hermetic: possibility to start multiple instances of them with different options.

Example: a pipeline for processing network data which consists of tcp endpoint and several intercommunicationg processes that consume data, we want to start different pipelines independently for different network endpoints (tcp ports).

There are several ways to do it

## Global Context Container

### Process Registration by well-known name

The easiest and most common way is simply to register your processes using global names and simply hardcode them in your models.

%TODO%: code example

Instead of implicit name registration you can use some custom DI container.
DI container is basically just a registry - a way to register and lookup dependencies (process pid) by some arbitrary id.

These includes: 
* Using well-known and reliable registry for instance Elixir.Registry
* Using some self-written registry which stores its state (process pid mappings) as GenServer State or in ETS

When Process starts, it registers itself in some DI registry. When its pid is required by other processes they just ask these DI registry for necessary dependency.

This solution is really the same as previos one. And its problem is rather obvious. It is chicken and egg dilemma: To call registry you need in turn to know its id.

All in all this solution is easy and common but it prevents us to run supervision tree multiple times.

## Process Registration by well-known name with prefix

To make our supervision tree hermetic we can prefix all processes with some common key and make it an init parameter of our core supervisor.

%TODO%: code example

## Implicit registry using Supervisor.which_children function

If we pay closer attention to Supervisor we can notice that any supervisor is basically a regitry itself.
It naturally knows pids of all its processes and all processes have uniq ids local to supervisor.
Unfortunately Supervisors are single purpose building blocks and therefore lack convinient apis to lookup there children. However we still can use supervisors as registers by calling.

We can exploit this fact by using Supervisor.which_children function.

%TODO%: code example

## Starting dependencies ad-hoc

%TODO%

%TODO%: code example

## Explicit local Registry

registry started outside of supervisor but linked to it

%TODO%

%TODO%: code example

## Explicit local Registry discovered by Supervisor.which_children

%TODO%

%TODO%: code example
