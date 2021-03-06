# Migrating to Diego

Diego is meant to be an in-place replacement of the DEAs.  Droplets staged on DEAs should run on Diego without any changes (and vice versa).

With that said, there are a handful of differences between Diego and the DEAs.  Some of these are slated to be addressed.  Some are not (though they can be if we get feedback that they need to be).

This migration guide is made up of three sections:

- [**Targeting Diego**](#targeting-diego) is intended for *developers* and describes the API calls necessary to run on Diego.
    + [Installing the `Diego-Enabler` CLI Plugin](#installing-the-diego-enabler-cli-plugin)
    + [Starting a new application on Diego](#starting-a-new-application-on-diego)
    + [Transitioning an application between backends](#transitioning-an-application-between-backends)
    + [Running route-less applications (such as workers and schedulers)](#running-route-less-applications-such-as-workers-and-schedulers)
    + [Recognizing capacity issues](#recognizing-capacity-issues)
- [**Diego Deltas**](#diego-deltas) describes known differences between Diego and the DEAs.
    + [Staging Performance](#staging-performance)
    + [Files API](#files-api)
    + [CF-Specific Environment Variables](#cf-specific-environment-variables)
    + [Disk Quota Over-Enforcement during Container Setup](#disk-quota-over-enforcement-during-container-setup)
    + [Health Checks](#health-checks)
    + [Behavior of Crashing Applications](#behavior-of-crashing-applications)
    + [Environment Variable Interpolation](#environment-variable-interpolation)
    + [File Permission Modes](#file-permission-modes)
    + [Mixed Instances](#mixed-instances)
- [**Managing the Migration**](#managing-the-migration) is intended for *operators* and describes the tooling available to manage a migration to Diego and proposes some approaches.
    + [The Importance of Communication](#the-importance-of-communication)
    + [Auditing Applications](#auditing-applications)
    + [Controlling Access to the Diego Boolean](#controlling-access-to-the-diego-boolean)
    + [Setting the Default Backend](#setting-the-default-backend)
    + [Forcibly Moving Applications](#forcibly-moving-applications)
    + [A Detailed Transition Timeline](#a-detailed-transition-timeline)

## Targeting Diego

App developers can ask CF to run their applications on Diego by setting the `diego` boolean field on their application to `true`.  Applications with their `diego` field set to `true` will both stage and run on Diego.

It is possible to modify the `diego` field on a running application.  This will cause it to transition from one backend to the other immediately, although without guaranteed uptime. To ensure uptime, we recommend performing a [blue-green deployment](http://docs.cloudfoundry.org/devguide/deploy-apps/blue-green.html) in which the new, 'green' app is placed onto Diego intentionally.

The following instructions assume you have the [`Diego-Enabler` CLI plugin](https://github.com/cloudfoundry-incubator/Diego-Enabler).  Instructions for installing it follow.

### Installing the `Diego-Enabler` CLI Plugin

The [`Diego-Enabler` CLI plugin](https://github.com/cloudfoundry-incubator/Diego-Enabler) makes opting into Diego easier. It is intended for use with CF CLI v6.13.0+. Install it from the CF-Community repo as follows:

```
cf add-plugin-repo CF-Community http://plugins.cloudfoundry.org/
cf install-plugin Diego-Enabler -r CF-Community
```

For CF CLI versions older than v6.13.0, install the [`Diego-Beta` plugin](https://github.com/cloudfoundry-incubator/diego-cli-plugin) instead.


The `Diego-Enabler` (and `Diego-Beta`) plugin includes subcommands to `enable-diego` and `disable-diego` for an app.  You can also check on whether an application has opted into Diego via `has-diego-enabled`.  There is also support around modifying the application's health check with `set-health-check` and `get-health-check`.

### Starting a new application on Diego

To start a new application on Diego you must push the application *without starting it*.  Once the app is created, you can set the `diego` boolean on it and *then* start it.

1. Push the application without starting it:

    ```
    cf push APPLICATION_NAME --no-start
    ```

2. Set the `diego` boolean:

    ```
    cf enable-diego APPLICATION_NAME
    ```

    This is equivalent to running `cf curl /v2/apps/$(cf app APPLICATION_NAME --guid) -X PUT -d '{"diego":true}'`

3. Start the application:

    ```
    cf start APPLICATION_NAME
    ```

### Transitioning an application between backends

Simply setting the `diego` boolean via

```
cf enable-diego APPLICATION_NAME
```

will cause an existing application to transition to Diego.  The application will immediately start running on Diego and will *eventually* stop running on the DEAs.  While this gives some safety, there are no strong guarantees around uptime.

If you want to ensure uptime we recommend performing a blue-green deploy (that is, push a copy of your application to Diego, then swap routes and scale down the DEA application).


To transition back to the DEAs, run

```
cf disable-diego APPLICATION_NAME
```

To tell which backend the application is targeting, run

```
cf has-diego-enabled APPLICATION_NAME
```

### Running route-less applications (such as workers and schedulers)

For the DEA backend, `cf push APP_NAME --no-route` does two things:

- it skips creating and binding a route for the application
- it implicitly causes the DEAs to skip the port health-check on application startup

> By default, when starting an application the DEAs wait until the application is listening on its assigned port *before* marking it as ready to receive traffic.  To determine whether or not to perform this check, the DEA inspects the routes bound to the application and determines: if they're present the port check is performed.  If they're empty, no port check is performed.

Diego configures its health checks [differently from the DEAs](#health-checks).  With Diego, `cf push APP_NAME --no-route` only skips creating and binding a route for the application.  It does not tell Diego which type of health check to perform.

By default, Diego does the same port-based health check that the DEA performs.  If your application does *not* listen on a port (for example, if you are pushing a worker or a scheduler app), then it will never satisfy this port check, and Diego will eventually mark it as crashed.  In these cases you must tell Diego not to perform a port-based health check via:

```
cf set-health-check APPLICATION_NAME none
```

The `none` name here is unfortunately misleading: if your app instance exits unexpectedly, Diego will still detect this and restart it automatically. We plan to add `process` as a better name for this type of health-check soon.

For the time being, the two valid values for the health check are currently `port` and `none`, with `port` the default. You can retrieve the current health check for your application via

```
cf get-health-check APPLICATION_NAME
```


### Recognizing capacity issues

The Cloud Controller is responsible for scheduling applications on the DEAs.  With Diego this responsibility shifts entirely to Diego.  As a result, the Cloud Controller does not know, ahead of time, whether or not there is capacity to stage or to run the application.  Instead, this information (referred to as a *placement error*) is available *asynchronously* and comes from Diego via the `cf app` API.

The CLI has already been updated to:

- display placement error information when `cf app` is invoked
- inform users when staging fails because of a placement error
- inform users when `cf push` fails because the application cannot be placed

> Currently, `cf apps` is misleading.  It will show all instances as healthy even if some of them have a placement error.  We intend to address this soon.

## Diego Deltas

Here's a list of some of the (known!) differences between Diego and the DEAs.

### Staging Performance

Diego's staging performance is somewhat slower than the DEAs.

##### Why?

The DEAs are tightly coupled to the notion that staging entails running through a set of buildpacks.  Because of this there are optimizations in place that treat buildpacks as *special things*: in short, the DEAs basically mount the buildpacks directly into containers.

Diego, being a generic container runtime, does not treat buildpacks in a special way.  They are simply assets that are downloaded and copied into containers.  The only optimization in place is a local download cache that allows Diego to avoid the download step.  At this time, however, the buildpacks need to be *copied* into each container - this copy step is expensive and does not parallelize well (it's disk-performance-bound).  This is exacerbated by the size of CF's offline buildpacks.

##### Workarounds

The simplest way to speed up staging performance is to specify a particular buildpack via the `-b` flag on `cf push`.  This will cause Diego to only fetch and copy in the single specified buildpack.  For example,

```
cf push my-app -b ruby_buildpack --no-start
cf enable-diego my-app
cf start
```

will stage the requested application using only the Ruby buildpack.

##### Future plans

There are plans to eventually improve Diego's semantics around mounting shared volumes across containers.  When this lands we will be able to directly mount the buildpacks into the containers (much like the DEAs do) without breaking Diego's generic abstractions.  A placeholder story is [here](https://www.pivotaltracker.com/story/show/88734100).

### Files API

Diego does not support the `cf files` API.

##### Why?

CF's existing feature set is massive and we had to cut scope to ship a Diego beta in a timely fashion.  Moreover, while supporting the files API in particular is relatively straightforward - we are hoping to solve the problem of "accessing a running container" more generically and comprehensively.

##### Workarounds

None*

> * Technically, there are hacky ways around this.  There's nothing stopping running applications from having secured endpoints that fetch files from their local filesystems...

##### Future Plans

We are planning on providing full-blown SSH access to running containers.  The intent is to support at least the following three usecases:

- Shell access via SSH
- `scp` support for fetching files
- Support for port-forwarding

We believe this solves a host of problems by building on top of an established, and secure, protocol.

### CF-Specific Environment Variables

Cloud Foundry supplies certain environment variables to app instances running on the DEAs, as documented [here](http://docs.cloudfoundry.org/devguide/deploy-apps/environment-variable.html). These environment variables differ slightly for app instances running on Diego.


#### `VCAP_APPLICATION`

There are a few entries in the `VCAP_APPLICATION` payload that are not provided on Diego:

- `users`: This value has apparently been `null` since some time in 2012.
- `started_at_timestamp` and `state_timestamp`: Time at which the instance is considered started, in Unix epoch time format.
- `started_at` and `start`: Same as `started_at_timestamp`, but in human-readable format.

Additionally, while Diego does now provide `application_uris` and its undocumented alias `uris` in the `VCAP_APPLICATION` payload, the values will always be stale if routes are mapped or unmapped form the app after its latest restart. On the DEAs, these values may be out of date, but will eventually converge as individual application instances restart. To guarantee that an app has the current list of URIs, it must be restarted via Cloud Controller (so that Diego receives a new DesiredLRP specification with updated `VCAP_APPLICATION` fields).


#### `VCAP_APP_HOST`

The DEAs currently set this environment variable to `0.0.0.0` in all cases.


#### `VCAP_APP_PORT`

This environment variable is deprecated. Apps should now use `PORT` or `CF_INSTANCE_PORT` instead.


##### Workarounds

- Unfortunately, Cloud Controller disallows users from setting the `VCAP_APP_HOST` environment variable on an app, or indeed any environment variable prefixed with `VCAP_`. It is recommended that you migrate away from the `VCAP_APP_HOST` environment variable, especially as it no longer provides useful information for the app instance.


### Disk Quota Over-Enforcement during Container Setup

When copying a droplet, a buildpack, or other assets into a container, the Garden-Linux backend may end up over-reporting the amount of disk used in that container. If this disk usage exceeds the quota allocated to the container, the copying-in operation will fail, and the container will crash. If you see crash events for your CF app with the exit description, "Copying into the container failed", this quota issue is likely the cause.

This erroneous reporting appears to be an interaction between the how the backing filesystem that garden-linux uses for container images accounts for disk usage and how payloads are streamed into the container. Once the payloads have been copied in successfully, the disk usage is eventually reported accurately (or even as less than expected, due to the backing filesystem's ability to de-duplicate some data in the files it stores).


##### Workarounds

Application developers can increase the amount of disk allocated to their application instances. As a rule of thumb, try allocating a disk amount at least twice the size of the unpacked application droplet (which can be determined by the disk usage reported when running on the DEAs).

To accommodate the resulting increase in the disk amounts allocated to instances, platform operators can allocate more disk to their cells, or can tune the reps to report more available disk than is actually present on the Cell VMs. This is effectively overcommitting disk on the Cells.
Platform operators may also need to increase the maximum allowed disk quota for an app instance in the Cloud Controller configuration, via the `cc.maximum_app_disk_in_mb` BOSH property in the CF deployment manifest, especially if apps over 1 GB in size are running on the deployment.

The Diego and Garden teams are still investigating the exact behavior and causes of this usage over-reporting, and would appreciate feedback, data points comparing droplet size with required minimum disk quota, and even test assets from the community as developers and operators encounter problems with the disk usage.


### Health Checks

The DEAs perform a single health check when launching an application.  This health is used to verify that the application is "up" before routing to it.  The default health check simply checks that the application has started listening on `$PORT`.  Once the application is up the DEA no longer performs any health checks.  The application is considered crashed *only* when it exits.  As mentioned [above](#running-route-less-applications-such-as-workers-and-schedulers) applications with no associated routes aren't health-checked at all.

Diego does health checks differently.  Like the DEAs, Diego performs the health check to identify when the application is "up".  Diego *continues* to perform the health check (every 30 seconds) after the application comes up.  This allows Diego to identify stuck applications -- applications that may still be running but are actually in a degraded state -- and restart them.

Currently Diego supports a port-based health check (like the DEAs).  However, Diego's health check is completely generic: Diego simply runs a process in the container periodically, and if the process exits succesfully the application is considered healthy.  There are plans to support URL-based health checks and, potentially, arbitrary custom health-check commands.

Applications that **do not** listen on a port will need to **disable** the health check.  This is described [above](#running-route-less-applications-such-as-workers-and-schedulers).

### Behavior of Crashing Applications

As with the DEAs/Health Manager, Diego restarts crashed applications.  There are a handful of differences:

- Diego does not keep crashed containers around.
- Diego *stops* restarting crashed applications eventually.

##### Why?

*Diego does not keep crashed containers around:*

With the advent of loggregator the need to keep crashed containers around (e.g. to fetch logs) is substantially reduced.

However, because Diego supports health checks it will be possible to identify containers that have "stuck" applications.  In that context having access to the container could help debug the broken application.  Once Diego enables SSH support we will consider allowing containers with unhealthy (but still running) applications to stick around (for an hour or so) to allow users to access and debug the stuck processes.

*Diego stops restarting crashed applications eventually:*

Our experience with large installations of CF is that there are a sizable number of applications that repeatedly crash and never fail to stay up.  (e.g. poorly written hello world applications).  These place a strain on the installation and consume unnecessary resources.  Diego attempts to restart these applications for approximately 2 days but then gives up on them.

##### Workarounds

None

##### Future Plans

Diego's restart policy is currently static.  There are plans to make it configurable on a space-by-space level.  This is discussed at length on [vcap-dev](https://groups.google.com/a/cloudfoundry.org/forum/#!topic/vcap-dev/tJTIkoD8__o/discussion) with stories beginning [here](https://www.pivotaltracker.com/story/show/87479698).

### Environment Variable Interpolation

Diego does not interpolate environment variables (i.e. referring to one environment variable in another via `$` will not work).  We'd like to see if this is, in fact, an issue for people.  If you have trouble because of this please reach out on cf-dev and we can look into workarounds/fixes.


### File Permission Modes

Diego does its best to preserve the permissions modes set on files it copies into its containers. This behavior is in contrast to the DEAs, which blindly change the permission modes of all the application files to `0744` when staging and running an app. The Cloud Controller and CF CLI are now also introducing changes to respect the permission modes of the CF user's files during `cf push`.

In most cases, this change in behavior will not affect how your application runs, as the buildpack itself is responsible for constructing or supplying the executable file with the correct permissions. If you are pushing a pre-built binary or other executable artifact and specifying the start command to run it directly, though, you should now make sure that the execute bit is set on the executable artifact. Likewise, if your application depends on the permissions modes of its other files, those modes should be set correctly on the local files before they are pushed.

Because of the differences in permission-mode behavior between Windows and Linux, when a developer pushes an app from a Windows workstation, the CF CLI will not specify the permission modes, and they will default to `0744`. As the vast majority the feedback we have received about these permission-mode differences has resulted from executable permissions not being set, we expect that this behavior will still resolve most of these issues.


### Mixed Instances

With the DEAs it is currently possible to create end up with instances that are configured differently from other instances of an application.  For example, it is possible to modify the start command then scale an application up.  The new instances will have the new start command whereas the old ones will not.

This is bad.

Should an old instance need to restart it will restart with the *new* start command!  This is almost certainly not what you intended!

Diego does not allow this behavior.

##### Why?

Diego is actually very opinionated about what can change about a running application.  Currently, only routes and the number of instances can be modified without restarting the application.  Any other changes (e.g., changes to environment variables, start commands, and bound services ) necessarily require a restart.

Diego does not orchestrate this restart for you - it leaves it to the user to bring up new instances and bring down old instances.  This is the safest way to correctly and safely transition applications between configurations without causing downtime.

##### Workarounds

Always use a green-blue deploy strategy when modifying anything about a running application.  If you'd like some instances of an application to have different configuration than other instances you should, instead, stage and deploy two different applications.

> Alternatively you can use the `INSTANCE_INDEX` environment variable to dynamically change your application's behavior based on its instance number.  This is not recommended.

##### Future plans

None


## Managing the Migration

This section is intended primarily for operators of Cloud Foundry: those tasked with deploying and managing a Cloud Foundry installation.

### The Importance of Communication

Transitioning from one backend to another safely is a substantial undertaking.  While the Diego team has worked hard to make Diego backward compatible with the DEAs a transition isn't as simple as flipping a switch.

We expect that most operators will want to deploy Diego Cells alongside the DEAs for some period of time.  This will give operators an opportunity to get familiar with Diego, and developers the opportunity to try their applications on Diego and submit feedback.  The goal during this time is to build confidence in the new platform and to suss out any unanticipated incompatibilities and issues.

Navigating this transition effectively is more about human communication than it is about technology.  We expect a typical transition plan might look something like this:

- Operators deploy a small set of Diego Cells alongside an existing DEA deployment.
- Operators advertise the presence of Diego with a subset of developers encouraging them to try pushing their applications (particularly staging/test versions) to Diego.
- After a cycle of feedback and monitoring operators might invite more developers to opt-in to Diego.
- With time, operators might inform all their developers that Diego will be replacing the DEAs at some point in the future.  Operators would set a deadline for developers to opt-into Diego.
- When the deadline arrives, Operators can identify developers that have not opted into Diego and reach out to them.
- At some point if there are still stragglers that haven't opted into Diego, Operators can revoke the ability of their developers to opt into/out off Diego and forcibly transition the remaining applications.

In this way the Operator can ensure that all applications are on Diego before performing a deploy that tears down all the DEAs.  A more detailed timeline is [outlined below](#a-detailed-transition-timeline).

The following sections describe the tools available to the Operator.

### Auditing Applications

Operators can identify applications that are/are not targetting the Diego backend by querying the Cloud Controller API:

```
cf curl /v2/apps?q=diego:true
```

and

```
cf curl /v2/apps?q=diego:false
```

### Controlling Access to the Diego Boolean

The `cc.users_can_select_backend` BOSH property controls whether or not none-admin users can modify the Diego boolean.

### Setting the Default Backend

The `cc.default_to_diego_backend` BOSH property determines whether *new* applications run on Diego or the DEAs.

### Forcibly Moving Applications

After auditing applications and identifying the set that are not running on Diego, operators can use the CC API to set the Diego boolean:

```
cf curl /v2/apps/APP_GUID -d '{"diego": true}' -X PUT
```

We recommend doing this in batches to monitor the load on the Diego backend as applications transition from the DEAs to Diego.

### A Detailed Transition Timeline

Putting these APIs together we can paint a detailed picture of the transition plan outlined above.  To make this concrete we'll cook up an arbitrary transition date: in our example operators will be turning off the DEAs in Mid-September.

- June:
    + Operators deploy Diego Cells alongside the DEAs
    + Operators give users the ability to select which backend their applications run on (`cc.users_can_select_backend=true`)
    + Operators set the default Backend to DEA (`cc.default_to_diego_backend=false`)
    + Operators send an e-mail to a subset of developers, encouraging them to use the Diego CLI plugin to opt into Diego and provide feedback
- July:
    + After a beta-period, operators send an e-mail to all developers that:
        + Informs developers about Diego and includes (links to) instructions for transitioning to Diego
        + Directs developers to provide feedback about Diego
        + Informs developers that DEA support will be removed on September 1st
    + Operators monitor Diego to ensure there are sufficient Cells to handle the load as Developers transition to Diego
    + Operators also monitor the DEAs and reduce the number of DEAs as demand drops
- August:
    + Operators audit all running applications and identify applications that are *not* running on Diego
    + Operators e-mail developers that have not yet opted into Diego (this may occur several times leading up to the transition deadline)
    + Operators set the default backend to Diego (`cc.default_to_diego_backend=true`)
- September:
    + The public transition deadline has arrived
    + Operators revoke the ability for users to select which backend their applications run on (`cc.users_can_select_backend=false`)
    + Operators notify the remaining holdouts that their apps are going to be migrated to Diego
    + Operators begin transitioning none-Diego apps to Diego
- Mid-September:
    + Operators finish transitioning none-Diego apps to Diego
    + Operators delete the DEAs, the transition is complete
