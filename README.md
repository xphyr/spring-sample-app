# spring-sample-app

The following repo can be used to test out Jenkins CI/CD pipeline within an OpenShift 4.x cluster. We will leverage a Jenkins install running inside OpenShift but the instructions should work for an external Jenkins server as well, with the [Jenkins Client Plugin](https://github.com/openshift/jenkins-client-plugin) installed.

## Setting up the Jenkins Server in OpenShift

To set up for this demo, we will be creating four projects in your cluster:
1. cicd - this will run the jenkins server
2. development - this will be the development project
3. testing - this will be the integration testing project
4. production - this will be the production instance of our app

```
oc new-project cicd --display-name='CICD Jenkins' --description='CICD Jenkins'
oc new-app jenkins-ephemeral
# use "oc status" to watch the deployment of the jenkins server and wait for it to complete
# use the "oc get route" command to record the URL for jenkins and record this for future use
oc get route
oc new-project development --display-name='Development' --description='Development'
oc new-project testing --display-name='Testing' --description='Testing'
oc new-project production --display-name='Production' --description='Production'
```

We need to give the Jenkins service account permissions to edit configurations in our target projects:

```
oc policy add-role-to-user edit system:serviceaccount:cicd:jenkins \
    -n development
oc policy add-role-to-user edit system:serviceaccount:cicd:jenkins \
    -n testing
oc policy add-role-to-user edit system:serviceaccount:cicd:jenkins \
    -n production
```

Since we will be using images stored in the internal OpenShift Cluster Registry we need to give the deployer in each of our testing and production projects permissions to pull from the development registry.

```
oc policy add-role-to-group system:image-puller system:serviceaccounts:testing \
    -n development
oc policy add-role-to-group system:image-puller system:serviceaccounts:production \
    -n development
```

## Setting up your development environment

Start by cloning the following github repo: https://github.com/xphyr/spring-sample-app and checkout a copy of the code local to your machine.

In your favorite code editor, update line 2 in the Jenkins file to point to YOUR github repo:

```
def templatePath = 'veermuchandi/spring-mvn-base~https://github.com/xphyr/spring-sample-app'
```

Also update the URI reference in the spring_boot_pipeline_bc.yaml to point to your github repo as well:

```
    git:
        uri: https://github.com/xphyr/spring-sample-app
```

Before moving onto the next steps, commit the changes above to your repo:

```
git commmit -am "updating to point to my repo"
git push
```

## Deploying the spring-boot application

Assuming that you have committed your changes as outlined above, we can now deploy the application.  Switch to the cicd project and create the buildconfig from the file you edited earlier:

```
oc project cicd
oc create -f spring_boot_pipeline_bc.yaml
```

Using the Jenkins URI you gathered from the Jenkins Setup instructions, log into Jenkins.  You should find a folder called "cicd", select that and you should now have a pipeline called "cicd/spring-sample-app-pipeline".  Select the pipeline, and then click "Build Now".

At this point, you should see a Jenkins job start. Depending on the speed of your cluster this build process may take a few minutes.

Once the build completes succesfully, we need to go add a route to make the applicaiton accessible to the outside world

```
oc project development
oc expose svc/spring-sample-app
oc get route
```

Take the route listed in your output and paste it into a web browser. You should get a hello world message from your application. Leave this page open, we will use it again shortly.

### Update the Pipeline configuration to poll the SCM.

1. Log into Jenkins using the URL from the Jenkins setup, and select your pipeline.
2. Select "configure"
3. Under "Build Triggers" select "Poll SCM"
4. Enter the following for the schedule to check every 5 minutes for updates
   `H/5 * * * * `
5. Click Save

### Making changes to your application

Now that we have a build pipeline set up, lets make a small change to the application and watch it build and deploy for us automatically.

In your favorite editor, open src/main/java/com/example/SpringSampleAppApplication.java and edit line 44 replacing "Hello World" with "Hello Galaxy" (or whatever astronomical unit you wish to use) and save the file.

Go ahead and commit this change to your repo `git commit -am "updating source" && git push` and then open your jenkins build pipeline and watch. Within 5 minutes a new build should kick off and your application will be updated. When the build completes refresh your application web page and see that the message has been updated.

### Bringing it all together

So now we have a pipeline that will watch for code changes and promote those changes in to our Development deployment, but what if we want to take this further? We can make this a multi-stage pipeline, promoting the code into a Staging environment, and finally after human approval, into Production. 

Edit your Jenkins file and remove two lines from the file (lines 94 and 190):
* "    /* - we will remove this later"
* "        we will remove this line later */"

this will enable the additional build stages, commit this code to your rep and watch your Jenkins job. Go ahead and commit your changes to the Jenkins file and push to github. Log into Jenkins and follow along as you have two new build stages.  Note that the pipeline will pause at the "Promote to Production" stage, looking for approval from you to push the code to production. Click approve and let the pipeline finish.

As part of the pipeline we create a staging route and a production route, go get each of these routes and open in new windows:

```
oc project testing
oc get route
oc project production
oc get route
```

For each route you got above you should now see a web page for "Staging" and one for "Production".  Go ahead and change the message in src/main/java/com/example/SpringSampleAppApplication.java one more time and commit/push your change and check to see that the change is propigated all the way through your new pipeline.