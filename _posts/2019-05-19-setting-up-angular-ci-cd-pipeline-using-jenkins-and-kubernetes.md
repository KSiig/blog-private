---
layout: post
title:  "Setting Up Angular CI/CD Pipeline, Using Jenkins and Kubernetes"
date:   2019-05-19 12:00:00 +0200
categories: docker devops
image: https://i.imgur.com/BMOkkom.png
ascent_image:
  background: url('https://i.imgur.com/BMOkkom.png') center/cover
  overlay: false
invert_sidebar: false
---

I’m no expert on the subject, in fact I’ve only known what DevOps is for about a year. This is what lead me to having a really tough time setting this up. The goal with this article, is to both explain how to set up, but also what problems I ran into. Seeing how things update so fast in this field, I feel it’s important not only to know, how things are set up, but to understand why things work the way they do.

# What Will This Article Cover?

I will do my best to split this up into sections. As the title of the article suggests, we will be using Jenkins on Kubernetes to deploy this CI/CD pipeline. As for testing Angular, we are going to be using Karma, although hopefully you will understand why things are working, by the end of this article, and use whatever testing framework you need.

# Preparing the Cluster with Helm and Tiller

First thing we need to do, is to prepare the cluster. We’re going to set up Helm and Tiller. If you’ve already done this, feel free to move on. In case you haven’t read any articles of mine before, here is how I structure it: First I will explain what needs to be done, and thereafter why it works.

## What to Do

First thing to do, is to download Helm. Go to the [official GitHub Releases](https://github.com/helm/helm/releases) page, and download the newest version. As of writing this, that would be v2.13.1.
>  **NOTE**: As of writing this, the makers of helm are starting to get serious about releasing helm 3. Helm 3 will be a big change, and most likely won’t be compatible with how helm 2 works. The rest of this article should still be the same, but do be aware of this!

Once you’ve downloaded helm, you should extract it, and move it your bin folder (usually /usr/bin for linux, /usr/local/bin for mac).

```bash
tar zxfv helm-v2.13.1-darwin-amd64.tar.gz
cd darwin-amd64
sudo mv helm /usr/local/bin
```

The commands above have been executed on a mac, however the procedure on linux is the same, merely with some filenames and directory names changed.

Next thing to do is to install Tiller, the component of helm that works on the server side. This takes three steps: Making a Tiller ServiceAccount, making a ClusterRoleBinding, and initialising Helm with the Tiller ServiceAccount. To create the ServiceAccount, copy the following into a serviceaccount.yaml file:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
```

After that, run kubectl apply -f serviceaccount.yaml. Now you should have a serviceaccount by the name tiller. This can be verified by running kubectl get serviceaccount tiller --namespace kube-system. After doing that, it’s time to create the ClusterRoleBinding.


```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```

Save the contents to a clusterrolebinding.yaml file, and run kubectl apply -f clusterrolebinding.yaml. At this point you should have a Tiller ServiceAccount, and have given it the privileges it needs to work. Now it’s time to initialise Helm, with the simple command helm init --service-account tiller. Hopefully by this point, you should have a working helm installation. Now for the important part of this section:

## Why it Works

If you haven’t worked with Helm before, that was quite a mouthful to go through. Let’s break it down now. The first thing we did, was to download Helm and make it so we can execute it on our system. I won’t go too much into that part. If it doesn’t make sense to you yet, you are going to have a tough time managing the rest of this project. [Here’s a great resource](https://eu.udacity.com/course/linux-command-line-basics--ud595) for learning some Linux basics.

Once we have Helm installed, we need to create two Kubernetes resources. I highly recommend you learn some basic Kubernetes before moving forward. Learn from my mistakes. When I tried setting all this up, I just tried learning Kubernetes on the go, and it caused me lots of confusion. [EdX has this](https://www.edx.org/course/introduction-to-kubernetes) really great resource for learning the basics.

So know that you, hopefully, know what ServiceAccounts and ClusterRoleBindings are, it’s time to understand exactly why we are doing all this. Helm is quite a big topic on its own, so here’s a brief explanation of it, and how it works together with Tiller.

Helm helps deploy applications on Kubernetes. It does this by what is called Charts. A so-called Chart has been developed for Jenkins, meaning we basically only have to run helm install jenkins. The concept may seem familiar, and it should. Just like apt is a package manager for debian, and brew is a package manager for MacOS; Helm is just a package manager for Kubernetes!

What helm does is act as the client side, and Tiller acts as the server side. Once you run the install command with Helm, it sends all the information needed, to a Tiller pod that lives in your Kubernetes cluster. Tiller then takes care of all the resources that are needed for whatever service you’re deploying. It also does health checks and so on, to make sure it’s always working. Enough about Helm and Tiller, let’s get on to installing Jenkins!

# Helm Install Jenkins

## What to Do

First you need to create a values.yaml file, and paste in the following content:

```yaml
master:
  installPlugins:
    - kubernetes:1.14.9
    - workflow-job:2.32
    - workflow-aggregator:2.6
    - credentials-binding:1.18
    - git:3.9.3
    - google-oauth-plugin:0.7
    - google-source-plugin:0.3
    - cobertura:1.13
    - blueocean:1.16.0
  cpu: "1"
  memory: "3500Mi"
  javaOpts: "-Xms3500m -Xmx3500m"
agent:
  enabled: true
  resources:
    requests:
      cpu: "500m"
      memory: "256Mi"
    limits:
      cpu: "1"
      memory: "512Mi"
persistence:
  size: 100Gi
networkPolicy:
  apiVersion: networking.k8s.io/v1
```

Then you simply run the following command:

```bash
helm install -n jenkins stable/jenkins -f values.yaml --wait
```

Once the command is done running, jenkins should be installed and working. Yes, with Helm it is actually as simple as that!

## Why it Works

In my opinion, this is really where the power of Helm shines! It completely works with the principles of Infrastructure as Code. There’s not much to break down here, besides the values.yaml file. This is something that is shared among almost all Helm charts.

The makers of whichever Helm chart you are using, have defined some values that you can give the installation. In this case, we are defining things, like what plugins Jenkins will install as default, some resource limitations, and some persistent storage. Having this file, means you can easily deploy the exact same version of Jenkins on a whole other cluster. The --wait flag means that Helm will wait until all resources are created, before it marks the installation as a success. By default, Helm will initialise the creating of all resources, and then finish the client side process.

# Setting up Jenkins

Now that all the stuff about Helm and installation is over, time to actually set Jenkins up!

## What to Do

Run the following two commands to get the ip address of your Jenkins server: 

{% raw %}
```bash
export SERVICE_IP="$(kubectl get svc jenkins --template {{ range (index .status.loadBalancer.ingress 0) }}{{ . }}{{ end }})"

echo [http://$SERVICE_IP:8080/login](http://$SERVICE_IP:8080/login)
```
{% endraw %}

Now you should have the ip address printed in your terminal, which you can then access through your browser. When you go to that url, you will see a login screen. With a default Helm install of Jenkins, the username is admin. The password can be printed by executing the following command:

{% raw %}
```bash
$(kubectl get secret jenkins -o jsonpath={.data.jenkins-admin-password} | base64 --decode)
```
{% endraw %}

Enter that into the login form, and congratulations! Now you have a fully functional Jenkins installation. One thing that’s not necessary to do, but that I recommend doing, is to update all plugins. This can be done by going to Manage Jenkins (can be found in the left menu) -> Manage Plugins -> Checking off checkbox for all plugins -> Download now and restart after install -> Set checkbox in ‘Restart Jenkins after install’. Once that is done running, you will now also have a completely updated version of Jenkins.
>  **NOTE**: The commands for getting the ip and password are also printed once helm has installed Jenkins. This information can be viewed at any time, by issuing the helm status <release-name> command.

## Why it Works

There’s not a lot of black magic behind this part, except for of course the ridiculous looking commands, to get the ip and the passwords. Let break them down.

{% raw %}
```bash
export SERVICE_IP=$(kubectl get svc jenkins --template {{ range (index .status.loadBalancer.ingress 0) }}{{ . }}{{ end }})

echo [http://$SERVICE_IP:8080/login](http://$SERVICE_IP:8080/login)
```
{% endraw %}

Exporting a variable and then using it, hopefully isn’t new to you. If it is, I suggest going back to my comment about learning Linux basics, before following this article. The interesting part is the kubectl command. Personally it started to make a lot more sense, once I viewed how the jenkins service is structured, if it’s seen in YAML.

```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2019-05-07T19:51:20Z"
  labels:
    app.kubernetes.io/component: jenkins-master
    app.kubernetes.io/instance: jenkins
    app.kubernetes.io/managed-by: Tiller
    app.kubernetes.io/name: jenkins
    helm.sh/chart: jenkins-1.1.11
  name: jenkins
  namespace: default
  resourceVersion: "3647856"
  selfLink: /api/v1/namespaces/medium/services/jenkins
  uid: 7c5519df-7101-11e9-8269-42010a9c005f
spec:
  clusterIP: 10.35.246.135
  externalTrafficPolicy: Cluster
  loadBalancerSourceRanges:
  - 0.0.0.0/0
  ports:
  - name: http
    nodePort: 31415
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app.kubernetes.io/component: jenkins-master
    app.kubernetes.io/instance: jenkins
  sessionAffinity: None
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
    - ip: X.X.X.X
```

The important part to look at in the command, is the following:

{% raw %}
```bash
{{ range (index .status.loadBalancer.ingress 0) }}{{ . }}{{ end }}
```
{% endraw %}

The range command stems from the fact, that Helm is currently using the Golang templating engine, where that comes from. In essence what it does, it define a range to work in. It makes a lot more sense, when you want to access more parts inside the same path, instead of just getting a single subject like we are here. Nonetheless; as can be seen in the YAML output above, there’s an entry called status containing a loadBalancer entry, and you see where this is going. The 0 in the command above, refers to the 0th index of the array, that is present within the ingress entry.

Now that that first pair of curly brackets has been defined, we want to get some specific info within that range. In this case it’s simply everything, since it only contains the ip we need. At the end there’s the fitting {{ end }} statement, simply another thing taken from the Golang templating engine.

The next command follows the almost the same logic, however it uses the jsonpath instead:

{% raw %}
```bash
$(kubectl get secret jenkins -o jsonpath={.data.jenkins-admin-password} | base64 --decode)
```
{% endraw %}

Jsonpath is a standard designed, as a way to get data out of a JSON object, in a well defined way. As stated, it follows most of the same logic as the command above, it’s just not as advanced.

# Setting up Jenkinsfile

Now that Jenkins is finally up and running, time to configure the pipeline we are going to use.

## What to Do

```groovy
pipeline {
  agent {
    kubernetes {
      label 'app'
      defaultContainer 'jnlp'
      yaml """
          apiVersion: v1
          kind: Pod
          metadata:
            labels:
              component: ci
          spec:
            serviceAccount: jenkins
            containers:
              - name: node
                image: signiant/docker-jenkins-nodejs  
                command:
                  - cat
                tty: true
        """
    }
  }
}
```

Copy the above code into a Jenkinsfile, placed in the root of your project. That’s really all there is to the functional part here.

## Why it Works

Getting the Jenkinsfile is as simple as copying the file above. Let’s get down to nitty gritty of why it works and what it does. What we’re using here, is referred to as a ‘Declarative Pipeline’ in the Jenkins community. There are other ways to set it up, which I won’t go into here.

First you need to define the pipeline block with the pipeline {} notation. Inside of here you can define various things, but for the time being, we’ll only focus on setting up the build agent. A build agent is a pod that Jenkins schedules in your Kubernetes cluster, with all the containers needed to build your app. In this case we are only using a single container, in order to build our Angular app.

Inside the kubernetes {} block, is where the actual containers are being specified. Things like label and defaultContainer are standard Jenkins terms, that help identify the build job. Inside the yaml part, you may notice that it’s just a simple Pod definition. So while the Jenkinsfile may seem big at first, it’s pretty manageable once you notice what parts it actually contains.

### The Containers

One of the absolute biggest problems I had, when setting up my Jenkins installation, was getting the right container. CircleCI provides some great containers for testing and building various frameworks, however because Jenkins work with some weird file permissions, they don’t work here. Unless you want to go into some really complex configuration in Jenkins, you need to have a container that’s running with the root user.

As you can see in the Jenkinsfile, I did end up finding a container that worked, but there is definitely a chance that that will end up not working as some point, so here are my experiences with what you need to do, in order to have a working container:

* It should be running as root user

* It needs to have Chrome 59+ installed, in order to run Headless

* Have Node installed

If you fill those three requirements, you should be able to get going.

# Setting up Karma

We’ll return to the Jenkinsfile later, first we’ll set up our karma configuration, so that it’ll work correctly with Jenkins.

## What to Do

First you need to install karma-junit-reporter so that Jenkins can look at the test results returned from karma. I’m personally using yarn, which then makes the command: yarn add -D karma-junit-reporter. When that is done, it needs to be added as a plugin in the karma configuration.

By default you will have a karma.conf.js file in the src directory. Open that and make sure you have the same plugins as below:

```javascript
plugins: [
      require('karma-jasmine'),
      require('karma-chrome-launcher'),
      require('karma-jasmine-html-reporter'),
      require('karma-coverage-istanbul-reporter'),
      require('@angular-devkit/build-angular/plugins/karma'),
      require('karma-junit-reporter')
    ]
```

Next step is to set up the correct reporter. change the coverageIstanbulReporter block corresponds to below:

```javascript
coverageIstanbulReporter: {
  dir: require('path').join(__dirname, '../coverage'),
  reports: ['html', 'lcovonly', 'cobertura', 'text-summary'],
    'report-config': {
        cobertura: {
            file: 'cobertura.xml'
        }
    },
  fixWebpackSourcePaths: true
},
reporters: ['progress', 'kjhtml', 'junit'],
junitReporter: {
        outputDir: 'test-results',
        outputFile: 'unit-test-results.xml'
    }
```

Notice that junit has been added to the reporters block, and a junitReporter block has been added. This defines where the test-results are outputted to.

Now that has been done, time to make a Custom Launcher. This is done by added a customLaunchers block, on the same level as the reporters block.

```javascript
customLaunchers: {
  ChromeHeadlessNoSandbox: {
      base: 'ChromeHeadless',
      flags: ['--no-sandbox']
  }
}
```

Now there’s only two things left to do. You will find a key called browsers with an array of strings. Change the array to contain a single string: ChromeHeadlessNoSandbox, the name of our custom launcher. Right below that you should find another key singleRun. Change that to true. Now you should have a file looking like the one below:

```javascript
// Karma configuration file, see link for more information
// https://karma-runner.github.io/1.0/config/configuration-file.html

module.exports = function (config) {
  config.set({
    basePath: '',
    frameworks: ['jasmine', '@angular-devkit/build-angular'],
    plugins: [
      require('karma-jasmine'),
      require('karma-chrome-launcher'),
      require('karma-jasmine-html-reporter'),
      require('karma-coverage-istanbul-reporter'),
      require('@angular-devkit/build-angular/plugins/karma'),
      require('karma-junit-reporter')
    ],
    client: {
      clearContext: false // leave Jasmine Spec Runner output visible in browser
    },
    customLaunchers: {
        ChromeHeadlessNoSandbox: {
            base: 'ChromeHeadless',
            flags: ['--no-sandbox']
        }
    },
    coverageIstanbulReporter: {
      dir: require('path').join(__dirname, '../coverage'),
      reports: ['html', 'lcovonly', 'cobertura', 'text-summary'],
        'report-config': {
            cobertura: {
                file: 'cobertura.xml'
            }
        },
      fixWebpackSourcePaths: true
    },
    reporters: ['progress', 'kjhtml', 'junit'],
    junitReporter: {
        outputDir: 'test-results',
        outputFile: 'unit-test-results.xml'
    },
    port: 9876,
    colors: true,
    logLevel: config.LOG_INFO,
    autoWatch: true,
    browsers: ['ChromeHeadlessNoSandbox'],
    singleRun: true
  });
};
```

## Why it Works

When working with unit testing, there are a few different standard reporters. Jenkins use the one called JUnit. That’s why we need to install the karma-junit-reporter package. Now it’s possibly to add a step to our Jenkinsfile, which Jenkins will recognise and give feedback on which test fail if any.

I won’t break down the entire coverageIstanbulReporter block. In essence what it does, is define what reports should be configured, and where the output for those reporters should be located.

The interesting part in my opinion is the Custom Launcher. Since Chrome 59, it is possible to run the browser in a headless mode. This is perfect for CI/CD scenarios. Since Jenkins isn’t running with a GUI per say, it would otherwise be impossible to run unit tests within Angular. The reason we are making a custom launcher, and not just putting ChromeHeadless as the browser (which is a possibility), is because of the security restricting Google has put in place.

Security wise, it’s almost never a good idea to run a browser as root, and Google will actually prevent anyone from doing that. That means, that if you are using it like we are, where you need to run it as root, you need to pass the --no-sandbox flag to Chrome when starting. This is exactly what using a Custom Launcher helps us with.

# Adding Stages

Now to adding the last step in the Jenkinsfile; stages. Everything in Jenkins is defined by stages, which then in turn contains steps.

## What to Do

```groovy
pipeline {
  agent {
    kubernetes {
      label 'app'
      defaultContainer 'jnlp'
      yaml """
          apiVersion: v1
          kind: Pod
          metadata:
            labels:
              component: ci
          spec:
            serviceAccount: jenkins
            containers:
              - name: node
                image: signiant/docker-jenkins-nodejs  
                command:
                  - cat
                tty: true
        """
    }
  }
  stages {
    stage('Test') {
      steps {
        container('node') {
          sh("yarn install")
          sh("yarn test")
          sh("yarn buildprod")
          junit 'src/test-results/**/*.xml'
        }
      }
    }
  }
}
```

Just like the first time you set up the Jenkinsfile, you just need to copy the above file. The only extra thing to do, is to modify the package.json file in your root directory. Add the two following scripts to the scripts section:

{% raw %}
```json
"buildprod": "ng build --prod",
"test": "ng test --no-progress --code-coverage"
```
{% endraw %}

## Why it Works

There isn’t much to this part. As can be seen in the Jenkinsfile, a Stage is added, which contains some certain steps. You can add as many stages as you want, with as many steps as you want.

The reason for adding the two script in the package.json file, is how the ng command works. If you just try to run ng build --prod in the container we are using, you will get an error message. ng is a package that is installed with npm or yarn into the project. Most developers can run it on their machine too, if they added the -g flag to the installation. However that hasn’t been done in the container, which is why we use yarn as a wrapper to run the commands.

# Setting up Project in Jenkins

Finally everything has been set up in the code, it’s time to set up the project in the Jenkins WebUI.

## What to Do

Open up Jenkins. On the far left side, press the ‘new item’ button. Then choose ‘Multibranch Pipeline’. Fill in the ‘Display Name’, which can just be the same as the project name. Then go down to Branch Sources. From here there are multiple sources to choose from. Pick the one where you are hosting your project. The UI should give enough guidance to get through this part.

Once that has been done, go down to ‘Scan Multibranch Pipeline Triggers’. If you want the project to be built periodically, check off the box, and choose what interval you want. If you don’t check off this box, you will have to manually start a build when you want it to run.

This section won’t include a ‘Why it Works’ section, considering how much it is just ‘point-and-click’ configuration. Once you’ve reached this point, you should have a fully functioning CI system running. The next part will be about setting up Cobertura. While Jenkins provides test coverage result, Cobertura provides even more information about test coverage.

# Setting up Cobertura

## What to Do

From the main page, go into the project you want to configure. In the left-side menu, choose ‘Pipeline Syntax’. In the middle of the page now, choose ‘cobertura’ as the sample step, then click ‘Generate Pipeline Script’ at the bottom.

Copy what it outputs, and insert it into your Jenkinsfile as a new step. Your ‘Test’ stage will then end up looking like so:

```groovy
stage('Test') {
  steps {
    container('node') {
      sh("yarn install")
      sh("yarn test")
      sh("yarn buildprod")
      junit 'src/test-results/**/*.xml'
      cobertura autoUpdateHealth: false, autoUpdateStability: false, coberturaReportFile: '**/cobertura.xml', conditionalCoverageTargets: '70, 0, 0', failUnhealthy: false, failUnstable: false, lineCoverageTargets: '80, 0, 0', maxNumberOfBuilds: 0, methodCoverageTargets: '80, 0, 0', onlyStable: false, sourceEncoding: 'ASCII', zoomCoverageChart: false
    }
  }
}
```

And that’s it. Nothing more to it. Whenever a build has finished, you will be able to look at a full coverage report by going to commit -> Coverage Report. At this point, a full CI system has been set up, with a detailed coverage report.

# Setting up Continuous Delivery

Luckily now that everything else has been set up, setting up the CD part of the pipeline is pretty much a piece of cake. There are only three parts that need to be done; setting up a new stage, setting up a new container, and adding some credentials.

## What to Do

Once again change your Jenkinsfile. This time change it to the following:

```groovy
def projectName = '<project-name>'

pipeline {
  environment {
    DOCKER = credentials("docker")
  }
  agent {
    kubernetes {
      label 'app'
      defaultContainer 'jnlp'
      yaml """
          apiVersion: v1
          kind: Pod
          metadata:
            labels:
              component: ci
          spec:
            serviceAccount: jenkins
            containers:
              - name: node
                image: signiant/docker-jenkins-nodejs  
                command:
                  - cat
                tty: true
              - name: docker
                image: gcr.io/cloud-builders/docker
                command:
                  - cat
                tty: true
                volumeMounts:
                  - name: dockersock
                    mountPath: /var/run/docker.sock
            volumes:
            - name: dockersock
              hostPath:
                path: /var/run/docker.sock
        """
    }
  }
  stages {
    stage('Test') {
      steps {
        container('node') {
          sh("yarn install")
          sh("yarn test")
          sh("yarn buildprod")
          junit 'src/test-results/**/*.xml'
          cobertura autoUpdateHealth: false, autoUpdateStability: false, coberturaReportFile: '**/cobertura.xml', conditionalCoverageTargets: '70, 0, 0', failUnhealthy: false, failUnstable: false, lineCoverageTargets: '80, 0, 0', maxNumberOfBuilds: 0, methodCoverageTargets: '80, 0, 0', onlyStable: false, sourceEncoding: 'ASCII', zoomCoverageChart: false
        }
      }
    }
    stage('Build') {
      steps {
        container('docker') {
          sh("docker login -u $DOCKER_USR -p $DOCKER_PSW")
          sh("docker build -t $DOCKER_USR/${projectName} .")
          sh("docker push $DOCKER_USR/${projectName}")
        }
      }
    }
  }
}
```

As you can see, there’s a new environment block, where we define an env variable containing our docker credentials. To add these credentials, open the Jenkins WebUI again. In the left-side menu, go to Credentials -> System. Then go to ‘global credentials’ in the middle of the screen, then click ‘Add Credentials’ in the left menu.

The kind of credential should be the default ‘Username with password’. Then enter your username and password for Docker Hub, and write ‘docker’ as the id, as you can see it defined in the environment block of the Jenkinsfile.

>  **NOTE: **if you host your containers somewhere else, you can enter those credentials too. Just make sure to change the ‘push’ step in the Jenkinsfile accordingly, so it matches your container repo.

Once that is saved, last thing to do is change the name of your project in the top of the Jenkinsfile. Once you’ve saved the file and committed it to your repo, Jenkins will pick it up, and build according to the new Jenkinsfile.

Congratulations! You now have a full CI/CD system set up!

## Why it Works

Hopefully this doesn’t seem all that far fetched for you. Except for adding some credentials, none of the concepts are new, compared to everything else that has been described in this article. If something doesn’t make sense, try reading the article again. Or if ends up being me who made a mistake, and didn’t write about it, please comment below and bring it to my attention.

# Final Thoughts

So now you have a basic CI/CD system set up, what next? In this article I’ve only described a basic setup, there are still a lot of things that can be improved. A lot of time is spent by pulling the images needed to build, every time Jenkins builds. Time could be saved there by caching the images in your cluster. The same is true for the node_modules.

Hopefully this can serve a starting point for building on your CI/CD system, and help you on your way!
