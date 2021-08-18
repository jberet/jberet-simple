# Overview

This is a sample standalone Java batch-processing application. The batch job
contains a single chunk-type step that reads a list of numbers by chunks and
prints them to the console. The 2 batch artifacts used in this application are:

* `arrayItemReader`: implemented in `jberet-support`, reads a list of objects configured in job xml

* `mockItemWriter`: implemented in `jberet-support`, writes the output to the console or other destinations

## How to Build

    mvn clean install

## How to Run Test Job

    mvn integration-test

## How to Run Application Main Class with maven exec plugin

    # run with the default configuration in pom.xml
    mvn exec:java

    # run with job xml
    mvn exec:java -Dexec.arguments="simple.xml"

    # run with job xml and job parameters
    mvn exec:java -Dexec.arguments="simple.xml jobParam1=x jobParam2=y jobParam3=z"

## How to Build and Run with `openshift` Profile

When building this application in OpenShift cloud platform, the `openshift` profile
defined in pom.xml will be used.  The difference between the default and `openshift`
profile is that the latter will package the entire application and its transitive
dependencies as an executable uber jar.

This process happens automatically when you build and deploy it to OpenShift
with `S2I`.  To build and run it locally with `openshift` profile:

    mvn clean install -Popenshift
    java -jar target/jberet-simple.jar simple.xml jobParam1=x jobParam2=y

## How to Build and Deploy to OpenShift Online

    # log into your OpenShift account
    oc login https:xxx.openshift.com --token=xxx

    # create a new project, if there is no existing projects
    oc new-project <projectname>

    # We wil use `openjdk18-openshift` image stream. Check if it is available in the current project
    oc get is

    # If `openjdk18-openshift` is not present, import it
    oc import-image my-redhat-openjdk-18/openjdk18-openshift --from=registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift --confirm

    # create a new application (with default name)
    oc new-app openjdk18-openshift~https://github.com/jberet/jberet-simple.git

    # or create a new application with custom name
    oc new-app openjdk18-openshift~https://github.com/jberet/jberet-simple.git --name=hello-batch

    # check status
    oc status

    # list pods, and get logs for the pod associated with the application:
    oc get pods
    oc logs jberet-simple-1-kpvqn

From the above log output, you can see that the application has been successfully built
and deployed to OpenShift online.  Alternatively, you can perform all of the above
operations in OpenShift online web console.

## How to Launch a Job Execution from OpenShift Command Line

Next, we will see how to launch the above batch job from OpenShift command line (`oc`).
First, create a yaml file (`simple.yaml`) to describe the Kubernetes job configuration:

```yaml

apiVersion: batch/v1
kind: Job
metadata:
  name: simple
spec:
  parallelism: 1
  completions: 1
  template:
    metadata:
      name: simple
    spec:
      containers:
      - name: jberet-simple
        image: docker-registry.default.svc:5000/pr/jberet-simple
        command: ["java",  "-jar", "/deployments/jberet-simple.jar", "simple.xml", "jobParam1=x", "jobParam2=y"]
      restartPolicy: OnFailure

```

Then, run the following command to tell OpenShift to launch the job execution:

```

/Users/cfang/dev/jberet-simple > oc create -f simple.yaml
job.batch "simple" created

/Users/cfang/dev/jberet-simple > oc get jobs
NAME      DESIRED   SUCCESSFUL   AGE
simple    1         0            4s

/Users/cfang/dev/jberet-simple > oc get jobs
NAME      DESIRED   SUCCESSFUL   AGE
simple    1         1            12m

/Users/cfang/dev/jberet-simple > oc get pods
NAME                    READY     STATUS             RESTARTS   AGE
intro-jberet-6-s94h2    1/1       Running            0          1d
jberet-simple-5-build   0/1       Completed          0          11h
jberet-simple-6-build   0/1       Completed          0          8h
jberet-simple-6-wwjm7   0/1       CrashLoopBackOff   105        8h
postgresql-5-sbfm5      1/1       Running            0          1d
simple-mpq8h            0/1       Completed          0          8h

/Users/cfang/dev/jberet-simple > oc logs simple-mpq8h > /tmp/simple-mpq8h.log

/Users/cfang/dev/jberet-simple > oc delete job simple
job.batch "simple" deleted


```

## How to Schedule Repeating Job Executions with Kubernetes Cron Jobs from OpenShift Command Line

First, create a yaml file (`simple-cron.yaml`) to define the Kubernetes crob job spec.
The cron expression `*/1 * * * *` specifies running the batch job every minute.

```yaml

apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: simple-cron
spec:
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: simple-cron
            image: docker-registry.default.svc:5000/pr/jberet-simple
            command: ["java",  "-jar", "/deployments/jberet-simple.jar", "simple.xml", "jobParam1=x", "jobParam2=y"]
          restartPolicy: OnFailure

```

Then, run the following commands to tell OpenShift to schedule the job executions:

```

$ oc create -f simple-cron.yaml
cronjob.batch "simple-cron" created

$ oc get cronjobs
NAME          SCHEDULE      SUSPEND   ACTIVE    LAST SCHEDULE   AGE
simple-cron   */1 * * * *   False     0         <none>          7s

# Get status of a specific cron job
$ oc get cronjob simple-cron
NAME          SCHEDULE      SUSPEND   ACTIVE    LAST SCHEDULE   AGE
simple-cron   */1 * * * *   False     0         <none>          24s

# Get continuous status of a specific cron job with --watch option
$ oc get cronjob simple-cron --watch
NAME          SCHEDULE      SUSPEND   ACTIVE    LAST SCHEDULE   AGE
simple-cron   */1 * * * *   False     0         <none>          33s
simple-cron   */1 * * * *   False     1         7s        46s
simple-cron   */1 * * * *   False     0         37s       1m

$ oc get pods
NAME                           READY     STATUS              RESTARTS   AGE
postgresql-5-sbfm5             1/1       Running             0          27d
simple-cron-1536609780-fmrhf   0/1       ContainerCreating   0          1s
simple-mpq8h                   0/1       Completed           0          26d

$ oc logs simple-cron-1536609780-fmrhf

$ oc delete cronjobs/simple-cron
cronjob.batch "simple-cron" deleted

# Another variation of the delete command:
$ oc delete cronjob simple-cron
cronjob.batch "simple-cron" deleted

```

## More info

[OpenShift Developer Guide](https://docs.openshift.com/online/dev_guide/jobs.html)

[JBERET-349](https://issues.jboss.org/browse/JBERET-349) Integrate JBeret with openshift scheduled job capability

[JBERET-448](https://issues.jboss.org/browse/JBERET-448) Move test-apps/simple to its own repo

[JBERET-449](https://issues.jboss.org/browse/JBERET-449) Support batch job execution with OpenShift cron job scheduling mechanism
