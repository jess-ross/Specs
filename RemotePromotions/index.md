# Remote Release Promotions

This is the living specification for the _Remote Release Promotions_ feature.

This feature will allow promoting a Release between Spaces.

!toc

## Concepts

- **Remote Environment**: An environment that lives in another Space. 
- **Release Bundle**: The promotion artifact. This will contain everything required to transport the release from one Octopus Space to another.   
- **Deployment Receipt**: A document representing the result of a deployment. This will be used to display the results of a remote deployment in the source Space. 
- **Connected Space**: The local and remote Spaces are able to communicate.
- **Disconnected Space**: The local and remote Spaces are isolated, and not able to communicate.  

## Implementation

### Remote Environments

Environments which live in another space will be able to be added to a Lifecycle. 

These remote environments will appear in the project dashboard. 

They will be able to have project variables scoped to them.

They will not able to have machines added to them, or permissions scoped to them. 

### Release Bundle

The release bundle will be a JSON document. It will contain everything needed to promote the release to a remote space. 

This will include: 

- Deployment Process 
- Variables
- Release details: Version, Notes, Package versions, Channel.
- Package references.

#### Packages
Packages will not be included directly in the bundle. They will sit beside. The bundle will reference the version and hash of the packages included.

The hash will be signed by the private-key of the local space.  This will ensure the exact same packages are used on the remote space.

#### Variables

Variables will be 'sliced' before being bundled. Any variables scoped to local environments (or machines) will not included in the bundle.

Sensitive-variables will be encrypted with the public-key of the remote space.

#### Schema Versioning
It would be a significant limitation if releases could only be promoted between spaces on the same version of Octopus.

To avoid this, the release bundle schema will be versioned.  Releases will be able to be promoted between spaces with matching schema versions.  The release bundle schema will hopefully rev far less frequently than Octopus versions.

### Promote a Release 

When a release reaches a point in it's Lifecycle where a remote environment is available, a 'Promote' action will be available.

e.g. In the image below, Release 1.1.0 is eligible to be promoted to Production.

![Promote Release from Dashboard](interface-designs/release-promote-accept/promote-dashboard.png "width=500")

### Accept a Release
![Accept Release Import](interface-designs/release-promote-accept/accept-release.png "width=500")

### Viewing the Deployment Process for a Release<a name="view-release-deployment-process"></a>

In Octopus today, when you view a project's deployment process you are viewing the latest version.  There is no way to view the process that was snapshotted with a particular release.  

This has always been a gap.  Remote Promotions simply highlights it. 
For example, imagine a project has two Remote Environments: _Staging_ and _Production_.  And two releases have been promoted, a patch release (say `1.0.1`) into _Production_, and a feature release (`2.0`) into _Staging_.
If you were to view the deployment process for the project in the Remote Space, would it show the process as it was at `1.0.1` or `2.0`? 

We are proposing to implement viewing (read-only) the deployment process as it was at the time of the release. This will be useful even in non-remote-promotion scenarios, but will be critical for remote promotions. 

### Project variables are read-only on the remote space

Project variables will be snapshotted with the release (just as today), and will not be editable on the remote space.

To enable contributing environment-specific variables on the remote side, we will implement environment variable templates (see below).

### Environment Variable Templates

In Octopus today, variables can be scoped to environments.  If you forget to scope a value for an environment (e.g. if a new environment is added), there is no warning. The variable's value may silently fall back to a wider-scoped (or non-scoped) value. 

We feel this is unacceptable for Remote Promotions. 

The environments in the remote space may not even be known by the person responsible for maintaining a project's variables. Even if they are known, they are owned by the remote space: environments may be added or removed, and having to inform the upstream project teams each time this occurs feels like the direction for the dependency graph. 

To solve this, we are proposing extending the behaviour of [variable templates](https://octopus.com/docs/deploying-applications/variables/variable-templates).

Variable Templates were implemented specifically for multi-tenancy, to allow a project to declare the variables that each tenant must define. We will do the same for environments.  When you define a variable template you will be able to specify whether it is required to be defined per:

- tenant 
- environment
- tenant and environment (i.e. a value must be supplied for every tenant-environment combination)

![Variable Template Editor](interface-designs/variable-templates/variable-template-editor.png "width=500")

When the release bundle is promoted to the remote space, this will allow Octopus to inform the user's in the remote space which environment and tenant specific variables must be supplied.

This should also address [this issue with template variables](https://github.com/OctopusDeploy/Issues/issues/2710).

### Explicit Null Variable Template Values

Customers have requested [optional variable templates](https://github.com/OctopusDeploy/Issues/issues/2709).

After considering it, our current thoughts are that templates are about preventing _accidental omission_ of variables. 

We are going to take the approach that it's OK to not supply a value, so long as the value is intentionally not supplied.

We can do this using a [null-object pattern](https://en.wikipedia.org/wiki/Null_Object_pattern), with a corresponding UI:

![Set Template Value Explicitly](interface-designs/variable-templates/set-template-value-explicit.png "width=500")

### Release channel is read-only on the remote space 

Viewing a project's channels suffers from the same issues as viewing the deployment process.  It is temporal. On the remote space we are interested only in the channel that the release was created for. This will be unable to be modified on the remote space.  

Variable scoping and deployment step scoping will occur when the bundle is created. 

The lifecycle on the remote side will be chosen at the time the release bundle is imported, removing the need for the channel to participate in that process.

### Project Triggers
Project triggers will be able to defined on either side (local or remote).

Triggers will not be included in the release bundle.

