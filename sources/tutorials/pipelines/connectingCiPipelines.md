page_title: Connecting Shippable CI to Shippable CD Pipelines
page_description: How to connect your CI to your pipeline
page_keywords: getting started, pipelines, quick start, documentation, shippable

#Connecting CI to Pipelines
Combining CI and Pipelines gives you a powerful end to end Continuous Delivery solution that can be up and running in minutes. The typical workflow is demonstrated in our [tutorial showing how to deploy a sample application](samplePipeline/).

This tutorial is focused on a specific scenario: how do you trigger your pipeline after your CI build succeeds?

Let us assume the following scenario: Your CI build runs unit tests and generates a Docker image, which you push to a Docker registry. Your pipeline takes this image, creates a service manifest, and then deploys it to a test environment.

<img src="../../images/pipelines/connectingCiPipelinesHow.png"
alt="connecting CI with pipelines" style="width:800px;vertical-align: middle;display: block;margin-left: auto;margin-right: auto;"/>


We're going to see how we can replace the question mark in the picture above with a trigger that will trigger a new version of myImage to be created, which will in turn trigger the manifest job and so on.

## Using runCI (recommended)
This method utilizes a runCI pipeline job to give you the same `OUT` capability in CI that several other pipeline job types already have. [(see docs for detailed description)](../../../pipelines/jobs/overview/)

* First, make sure that your enabled project has an associated runCI job.  Today, runCI jobs are created automatically when a project is newly enabled for CI. To create a runCI job for a project enabled before Feb. 18, 2017, click the "Hook" button on the project settings page.  [See here](../../pipelines/jobs/runCI.md) for details about creating and utilizing runCI jobs.

If you see a job in your pipeline SPOG named after your CI project, with a `_runCI` suffix, then your project is ready to go.

It will look something like this:<br>

<img src="../../../pipelines/images/jobs/runCIInSPOG.png" alt="SPOG with runCI job" style="width:800px;vertical-align: middle;display: block;margin-left: auto;margin-right: auto;"/>


* Next, add this runCI job to your `shippable.jobs.yml`, specifying the image resource name that you want your CI to update.  Make sure the job name matches the one that was automatically created in your SPOG.  Note that you can specify multiple image resources, or any other kind of resource that you want to update during CI (triggers, params, files, etc).
```yml
jobs:
  - name: my_app_runCI
    type: runCI
    steps:
      - OUT: my_image

```

Just like that, your CI project is connected to your pipeline. It should look like this:<br>
<img src="../../../pipelines/images/jobs/runCIOutSpog.png" alt="SPOG with runCI job" style="width:800px;vertical-align: middle;display: block;margin-left: auto;margin-right: auto;"/>


* Finally, to signify a change in the resource, write a payload of your choice to the `<resourceName>.env` file during your CI build.  A change in the `.env` file will trigger a POST later on that will update your resource version and start subsequent pipeline jobs.
  * Every CI build has access to certain environment variables that represent the IN/OUT resources.  In this example, we'll be using the one that points the job's state directory.  You can [read here](../../pipelines/jobs/runSh.md) about the different variables we provide.
  * The `.env` file expects key/value pairs in the format of `<key>=<value>`. Usage of these files is described [here.](updateResourceVersion.md)
  * All `OUT` resources, as well as the running job (runCI), will have a `.env` file that you can update.

```yml
build:
  ci:
    - ./runTests.sh
    - ./buildAndPush.sh
    - echo "versionName=$BRANCH.$BUILD_NUMBER" >> $JOB_STATE/my_image.env
```

With this configuration, when you run a build you should see some extra logs that signify the updating of your resource. `versionName` is a standard property that exists for all resources, but you can create your own custom key/value pairs as well.  For image resources, the `versionName` represents the image tag. [See our resources page](../../pipelines/resources/overview/) for other special uses of `versionName`.

<img src="../../images/pipelines/outConsole.png" alt="Sample job log output for version updating" style="width:800px;vertical-align: middle;display: block;margin-left: auto;margin-right: auto;"/>

When the POST occurs for the new version of the resource, any subsequent jobs that depend on that resource will automatically be triggered.

## Using Event Triggers via notification integrations (deprecated)
NOTE: This method is not recommended, but will still work until we fully sunset the eventTrigger integration support.  We recommend you use the runCI method listed above to accomplish your scenarios instead.

* Create an API token for your account. To do this, go to your **Account Settings** by clicking on the gear icon in the top navbar. Then click on **API tokens** in the left sidebar menu and create a token. Copy the token since you won't be able to see it again.

* Next, create an account integration of type 'Event Trigger'
    * While still on **Account Settings**, go to  **Integrations** in the left sidebar menu and then click on **Add Integration**
    * Click the **Create Integration** button next to **Event Trigger**. In the **Create Event Trigger Integration** tab, complete the settings as shown below. Select the subscription containing your CI project; this will add the account integration to that subscription as a subscription integration. Please make sure you update the `Authorization` textbox in the format `apiToken <token-value>`. The resource name should be the resource that refers to the image you're pushing as part of your CI.

    <br>

<img src="../../images/pipelines/samplePipelineEventTriggerIntegration.png" alt="Shippable Continuous Integration and Delivery" style="width:800px;"/>

* Next, add the following to the shippable.yml for your CI project:

```
integrations:
  notifications:
    - integrationName: triggerDemoPipeline #Replace with name of the integration from subscription settings
      type: webhook
      payload:
        - versionName=$BRANCH.$BUILD_NUMBER  #Replace with the tag of the image you pushed during CI
      on_success: always
      on_failure: never

```

And that's it. The next time you run your CI build and push your image, your pipeline will be triggered by your event trigger integration.

<img src="../../images/pipelines/connectingCiPipelines.png" alt="connecting CI with pipelines" style="width:800px;"/>
