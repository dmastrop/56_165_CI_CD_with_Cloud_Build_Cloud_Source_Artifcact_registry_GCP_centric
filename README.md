This is a complete CI/CD pipeline implementation using GCP (Google) platform. There is no github actions, Jenkins, travis CI involved.

The three major components involved are the Google Artifact Registry (like docker hub), Google Cloud Build (to stage the CI and CD jobs and configure the relevant triggers based on the cloudbuild yaml scripts), and Google Cloud Source
Note that the Cloud Build SA (service account) must be given Kubernetes Admin role permission and write/push permissions to Cloud Source for this implementation to work.

There are 2 local repos that are syched to two corresponding Cloud Source repos (similar to github repos).  The first repo has the source code, the Dockerfile, a kubernetes manifest template folder and the cloudbuild.yaml file (which has all of the CI scripts for Cloud Build to execute).  Cloud Build has a GUI menu to configure the basic nature of the trigger, where the cloudbuild.yaml file is, etc for the CI phase.  Same thing is done for a second trigger for the CD phase (using a cloudbuild-delivery.yaml configuration file)

During CI stage if there is any change to the source code and if it is pushed to the Cloud Source repo that has the source code (first repo),  Cloud Build SA root user builds the new docker image and pushes it to the Google Artifact registry with the SHA tag based on the last commit id.  The next stage (3 steps indicated below) for the CI will **trigger** the CD (deployment) stage (it is not the actual deployment but triggers it; this is based on the deployment trigger configured through the Google web console as noted above).  

There are 3 steps in this next stage for CI: ((As stated above: There are 2 local repos that are sychned to two corresponding Cloud Source repos (similar to github repos).  The first repo has the source code, the Dockerfile, a kubernetes manifest template folder and the cloudbuild.yaml file (which has all of the CI scripts for Cloud Build to execute).))

Once the new docker image is pushed to the Google Artifact registry, the 3 following CI steps do this: first the Cloud Build clones the second Cloud Source registry locally (Cloud Build local environment) (this repo, a deployment Cloud Source repo will have the updated Kubernetes.yaml manifest for deployment of the new image,  and the cloudbuild-delivery.yaml which has the Cloud Build job scripts to actually deploy the image to GKE).

Second (CI-CD trigger phase), Cloud Build generates the new kubernetes manifest (Kubernetes.yaml) based on the template that it has (Kubernetes.yaml.tpl). It basically inserts the google project id and the tag of the latest Google Artifact registry image (that has the changes). This image is already on Google Artifact registry (see above)

Third (CI-CD trigger phase),  Cloud Build then pushes the new Kubernetes.yaml file to the second deployment Cloud Source repo. This push is configured in Cloud Build to trigger the CD (the actual deployment) (this is a second trigger that is configured in Cloud Build through the Google web console)

CD phase:  Once the new Kubernetes.yaml file has been pushed to the second Cloud Source deployment repo (this is pushed to the candidate branch initially, and pushed by the Cloud Build SA), the second trigger for CD starts. Cloud Build will then kubectl apply -f Kubernetes.yaml file to actually deploy this new build to the GKE as a running cluster.  (The cluster infra needs to be in place before this happens. There are a wide variety of tools to do this.  For amazon I use eks-ctl or kops to set up the infra and cluster; more advanced tools like kubeadm/terraform allow for much greater customization of the k8s cluster via the cluster management plane, if required, but this requires a lot of k8s management plane and software component knowledge)

The Cloud Build then pushes the Kubernetes.yaml file to the production branch of the second (deployment) repo on Cloud Source.  So, at the end of this process the candidate branch of the second repo will have the cloudbuid-delivery.yaml file (with the deployment job scripts; this does not change) and a Kubernetes.yaml file that changes with each new CI push of the updated image.  The production branch will also have the same. 

Ideally there needs to be a test stages also added to actually test the code (instead of just manually pulling up the browser).  These have not been implemented into the Cloud Build yaml script files yet. Various test phases should be inserted throughout the process. During the CI phase perhaps a static code analysis insuring that the code syntax runs without error or if compilation is required that the code compiles ok. If nodejs, there are many ways to implement this via the package.json scripts and using nodemon, the insertion of code is dynamic. Unit tests can also be added during this phase.

During the CD phase an integration test, performance test and ultimately a UI (selenium) test can be performed prior to the final push of the Kubernetes.yaml file to the production branch of the second (deployment) repo on Cloud Source. In this way, the production branch will only have the Kubernetes.yaml if the code has completed all stages of testing, inferring that the code (for this simple setup, it is just a docker file for the container that runs in the k8s cluster) referred to in the k8s manifest is production ready (performance and integration and UI are all passing). 

So there needs to be a series of testing suites between the candidate rollout to GKE and the final push to the production branch.  Once the production branch has the final Kubernetes.yaml a sanity test should be done, with the Cloud Build running the kubectl apply -f on the yaml to ensure that this comes up fine in GKE and a simple sanity selenium based test is performed on the UI.  If integration or UI tests fail during the candidate rollout, the process flow needs indicate failure, and the development teams need to be notified.  They can rollout changes in the code to kick off the new CI process again and start the process over again until the integration and UI tests pass for production quality.  If performance tests fail a decison tree can be implemented depending upon the severity of the performance deficit.  Not all performance deficits would require an unconditional code revision by development.
