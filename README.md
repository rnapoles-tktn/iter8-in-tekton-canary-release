# Performing Automated Canary Releases Using Iter8 In Tekton

In this article we look at using Iter8 in a Tekton pipeline to perform automated canary releases of a microservice.  With canary releases, a new version of your microservice is deployed to sit alongside your existing version and traffic is routed to your new version as specified in a set of rules.  You then monitor how your new version performs and make decisions about increasing the traffic to your new deployment until such a time that all traffic is directed to the new release. 

Iter8 v0.1.0 automated cloud-native canary releases, driven by analytics based on robust statistical techniques. It comprises two components, the iter8-analytics component and the iter8-controller component. 

The analytics component assesses the behavior of different versions of a microservice by analyzing metrics to determine if the newer version performs to the specified success criteria. 

The iter8 controller adjusts traffic between the different versions of the microservice based on recommendations from the analytics components' REST API.  Fundamentally, traffic decisions are made by the analytics component and honored by the controller. 

The test and its success criteria are defined in an experiment - a kubernetes custom resource defined by iter8.  In the experiment we define properties such as, the baseline and candidate deployments, the metric to be analysed (e.g iter8_latency: the latency of the service), the number of iterations of the experiment to run and what routing rules to apply, for example - increase traffic to the new service by 20% after each successful iteration.  The metrics that are analysed come from interrogations of prometheus by the analytics component.


## Overview Of This Tutorial

In this tutorial we have a simple web application made up of a front end service which performs a call to a backend service and displays the result to the user.

![Initial Application Flow](./images/application.png?raw=true "View Of The Two Services Communicating")

We will be making a change to the deployment yaml stored in a git repository, causing a webhook to trigger a Tekton pipeline that will create and run the Iter8 experiment.  We will generate web traffic to the application within the pipeline itself, faking user requests (as this is not a live application with actual users) for iter8 to analyse.  Iter8 will make use of istio to control the split of traffic between the older and newer deployments.

![Iter8 Experiment Flow](./images/experiment-architecture.png?raw=true "View Showing Iter8 Components with Application")

The pipeline we will use in the example will run through 5 tasks, creating the iter8 experiment, generating load to the application, applying the new deployment causing the experiment to start, wait for experiment completion and finally update the git commit status based on the result.

![Pipeline Overview](./images/pipeline.png?raw=true "Overview of Steps in The Pipeline")

The result of the experiment will dictate which deployment of the microservice is left running.  We will run through the code changes twice, once testing a deployment of the microservice that does not not meet the success criteria and then again with a version that does.

More on the Experiment CRD can be found in the iter8 [docs](https://github.com/iter8-tools/docs), but ensure you look at the relevant version of the documentation for your iter8 installation.

The Experiment as defined in this sample's pipeline Task:

```
      apiVersion: iter8.tools/v1alpha1
      kind: Experiment 
      metadata:
        name: ${EXPERIMENT_NAME}
      spec:
        targetService:
          name: $(inputs.params.service-name)
          apiVersion: v1 
          baseline: ${BASELINE_DEPLOYMENT_NAME}
          candidate: ${CANDIDATE_DEPLOYMENT_NAME}
        trafficControl:
          strategy: check_and_increment
          interval: 30s
          trafficStepSize: 20
          maxIterations: 8 #default value = 100
          maxTrafficPercentage: 80 #default value = 50
          onSuccess: ${ON_SUCCESS}
        analysis:
          analyticsService: "http://iter8-analytics.iter8:8080"
          successCriteria:
            - metricName: iter8_latency
              toleranceType: threshold
              tolerance: 0.2
              sampleSize: 6
```

The practical example will give you a flavour for how you can add iter8 into tekton pipelines.  With an understanding of this simple scenario you should be able to extrapolate how you can user Iter8 in a broader gitops scenario, such as: 

![Larger Conceptual Use of Iter8 In GitOps](./images/concept.png?raw=true "Overview of a Larger GitOps Solution Involving Iter8")

Here, iter8 is integrated into three pipelines across two repositories.  When developers submit pull requests (PR) to the microservice, iter8 is used as one of the verification tests - this would likely be in an isolated namespace so the base deployment of the master branch would need installing as well as the newly built service.  The results of the experiment here could be fed back and marked onto the pull request giving the developer the earliest possible opportunity to realise they have introduced a performance problem.

After an approver merges the pull request into the master branch another pipeline is triggered which runs an iter8 experiment on the code as it now appears in master - we could even use tools like odo within our pipeline to create pull requests into our gitops repo upgrading the image tag for the specific service upon succesful completion of all tests.

The gitops repo itself could trigger a third pipeline which looks to test the new build in the live environment.  If succesful iter8 can leave the new deployment in place and remove the old one, again tools like odo could be used to automerge the pull request into master from within the pipeline and also close out the pull request.  Notifications can even be sent to slack channel or emailed to advise of problems/successes.

From v1.0.0 of iter8 you will also be able to perform a/b deployments, but that is beyond the scope of this article.

See [here](INSTRUCTIONS.md) for instructions on running the sample.