Pipeline for Pivotal Redis deployments
======================================

![flow](https://github.build.ge.com/industrial-cloud-p-redis/cloud-foundry-deployments/raw/pipeline/docs/pipeline-flow.png)

Local development of templates
------------------------------

99% of the time - hopefully 100% of the time - you will only modify templates inside the `try-anything` environment. The pipeline will test your changes and then propogate them out to the other environments.

For any environment you can confirm that its current templates combine together into a manifest:

```
./environments/bosh-lite/try-anything/bin/make_manifest.sh
```

Adding new environment to pipeline
----------------------------------

Consider adding a new environment `sandbox0` to `AWS`.

First, bootstrap the folder space and the currently required local templates:

```
$ ./bin/new_environment.sh aws sandbox0
Copying in initial scripts
Copying in some example pipeline templates
Copying in some example release/stemcell versions
Copying in some example local environment templates
```

This will base the new environment off the `environments/bosh-lite/try-anything` templates.

Alternately, provide the relative path of the environment that the new environment will be similar to:

```
$ ./bin/new_environment.sh aws sandbox0 environments/aws/sandbox
```

Now populate these files with configuration for this new environment.

Next, add the new environment into the pipeline.

To confirm that your templates merge together ok, run:

```
./environments/aws/sandbox0/bin/make_manifest.sh
```

New environments for existing deployments
-----------------------------------------

If you already have an existing deployment then this section describes how your new environment in the pipeline can take over the existing, manual deployment.

**The goal is: make the first live deployment exactly the same as production.**

After the pipeline robot deployer can reproduce what is in production, then you can iterate towards the desired deployment via the new pipeline.

First, download all the live deployment manifests from the target BOSHes.

First, update the `name.yml` to specify the existing deployment name, say `p-redis-sandbox0`:

```yaml
---
name: p-redis-sandbox0
meta:
  environment: p-redis-sandbox0
```

Next, add the API/username/password credentials into `credentials.yml`:

```yaml
bosh-aws-sandbox0-target: https://10.10.10.2:25555
bosh-aws-sandbox0-username: admin
bosh-aws-sandbox0-password: PASSWORD
```

Run the following helper to fetch all the latest deployment manifests:

```
./bin/extract_latest_manifests
```

It will be stored in the environment's folder as `<deployment-name>.yml`, e.g. `environments/aws/sandbox0/p-redis-sandbox0.yml`.

Based on the copied templates from above, you can create a new `manifest.yml` - albeit it will look like the source environment and not yet look like the existing deployment.

```
./environments/aws/sandbox0/bin/create_stub_make_manifest_and_save.sh
```

This is the script that the pipeline will run - and it stores the manifest in `environments/aws/sandbox0/manifests/manifest.yml`.

Now show the difference between the current deployment manifest and your initial pipeline-based manifest:

```
spiff diff \
  environments/aws/sandbox0/p-redis-sandbox0.yml \
  environments/aws/sandbox0/manifests/manifest.yml
```

Building/updating the base Docker image for tasks
-------------------------------------------------

Each task within all job build plans uses the same base Docker image for all dependencies. Using the same Docker image is a convenience. This section explains how to re-build and push it to Docker Hub.

All the resources used in the pipeline are shipped as independent Docker images and are outside the scope of this section.

```
fly -t bosh-lite configure -c ci_image/pipeline.yml --vars-from credentials.yml p-redis-pipeline-image
```

This will ask your targeted Concourse to pull down this project repository, and build the `ci_image/Dockerfile`, and push it to a Docker image on Docker Hub.
