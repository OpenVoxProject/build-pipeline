# Community Packaging

Every bit used except for the Overlook InfraTech private key is available in public repos in the `overlookinfra` github org.
You'll notice it currently has a lot of `overlookinfra` nomenclature in here, but these will be changed to `OpenPuppet`
as soon as we've come to a firm naming decision.

## `puppet-agent`

Building off of [the work Jeff Clark did to improve Docker support in Vanagon](https://github.com/ospuppet/vanagon),
we [added a bunch more things](https://github.com/overlookinfra/vanagon/commit/f439446bfc885bde34da8d3b32f28ab1f72bd6d3)
to make building these packages with containers work better. You can see the choices of container images for each platform there.
We also tried to standardize the platform defaults a bit since they've gotten rather divergent over the years.
This also allows us to build for different architectures.
The packages are built all of this on [@nmburgan](https://github.com/nmburgan)'s Ryzen 5 5950X, and aarch64 and ppc64le were run under emulation with qemu.

In some cases, Perforce uses their own internal version of build tools (pl-build-tools) to build for older platforms
who ship with build tools that are tool old.  Rather than do this, we've moved utilizing publicly available updated
tools instead. These days, that seems to be mostly for el7.
.
In each of the component repos ([puppet-runtime](https://github.com/overlookinfra/puppet-runtime),
[pxp-agent-vanagon](https://github.com/overlookinfra/pxp-agent-vanagon), [puppet-agent](https://github.com/overlookinfra/puppet-agent)),
you'll see we have a `plumbing` branch, which is used for syncing the branches from `puppetlabs exactly` as they are.
The packaging code for each project is also going into this branch.

Each has an `overlookinfra.patch` which changes some of the component files in order to make them work correctly with
containerized building with our vanagon changes.

* **Rake tasks**
  * `Tag` - This checks out the given puppetlabs tag we want to build for, then applies the overlookinfra.patch, and then retags
      in the form `<puppetlabs tag>-overlookinfra`.  It pushes a branch with a name of the form `overlookinfra/<puppetlabs tag>` that
      contains the commit that is tagged.  In pxp-agent-vanagon and puppet-agent, this also overwrites the `component json` file
      to point to the [OSL puppet-artifacts S3 bucket](https://artifacts.overlookinfratech.com).
  * `Build` - This utilizes the `build-vanagon.rb` script which is also found in the `plumbing` branch.
      This is a pretty hacky script, but what it does is create directories for however many platforms you want to build in parallel,
      and then run those builds in the container for each platform.  You can certainly do 1 single thread to do one platform at a time.
      On my 32 hardware thread/32GB RAM system, 8 threads seems to be pretty reasonable.  All of the packages end up back in this
      repo in the `output` directory like it would with a normal vanagon build.
  * `Upload` - This is written specifically for uploading to the OSL S3 buckets.  You won't be able to use this without the
      secrets, but all the code is there.

First, `puppet-runtime` is built and uploaded to the `puppet-runtime` S3 bucket.  Then `pxp-agent`, which utilizes `puppet-runtime`, then `puppet-agent` which utilizes both.

## `puppetserver/puppetdb`

This has similar build and upload rake tasks as above.  Right now, we aren't needing to make changes to the repo for our purposes,
as we can use puppetlabs' ezbake for now, so no tag task.

Build uses the `build-ezbake.rb` hacky Ruby script. This creates (or reuses) [an image](https://github.com/overlookinfra/puppetdb/blob/plumbing/Dockerfile)
that is based on almalinux 9 that has Ruby, Lein, and Java installed. It then builds for all the platforms, since you don't
actually need to build on the specific platform your are building a package for.

The code used for signing the packages can be found at https://github.com/overlookinfra/misc/tree/main/signing. 
This uses an alma 9 container to sign RPMs and debian 12 to sign DEBs.  For debs, we use `aptly` for repo creation,
since creating a deb repo by hand is not so fun.  For the yum repo, it's a lot simpler. 
We sign with our GPG-KEY-overlook key, and the pub key is available in repos.

To create the repository packages (i.e. the rpm files at https://yum.overlookinfratech.com/ to set up the repo on your machine),
we used the overlook branch of [puppetlabs-release](https://yum.overlookinfratech.com/).  This is modified to point to the OSL
repos, and includes the pub key which gets placed and imported automatically.

### Disclaimer

This is basically all spike code, as we were trying to get something up and going as fast as possible, and a whole lot of copied
code between repos.  We would love some help in turning this into proper tooling (or perhaps integrating it into existing tooling)
and creating proper pipelines. We also don't have Windows or MacOS packages built out, as that's a bit of a heavier lift.

For what we have now, it's all built it all on home computers, and you should be able to as well for reproducibility purposes
except for the upload and signing part, since you'd need secrets for that.
