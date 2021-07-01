#!groovy

/*
 * This pipeline will copy a product form one 3scale Hosted instance to another
 * and create a new 3scale application. 
 * Setup instructions:
 * 1. Spin up a Jenkins master in a fresh OpenShift project
 * 2. Create a secret containing your 3scale_toolbox remotes:
 *
 *    3scale remote add 3scale-saas "https://123...456@MY-TENANT-admin.3scale.net/"
 *    oc create secret generic 3scale-toolbox --from-file="$HOME/.3scalerc.yaml"
 *
 * 3. If you chosed a different remote name, please adjust the targetInstance variable
 *    accordingly.
 * 4. Create a new Jenkins pipeline using the content of this file.
 */


def sourceInstance = "3scale-instance"
def targetInstance = "3scale-instance"
def sourceProductName = "hello_world_product"
def targetProductName = "hello_world_product_4"
def sourceAppPlanName = "hello_world_app_plan"
def targetAppPlanName = "hello_world_app_plan_4"
def sourceBackendName = "hello_world_backend"
def targetBackendName = "hello_world_backend_4"
def targetApplicationName = "hello_world_application_4"
def account = "test"


node {
  stage("Export product") {
    runToolbox([ "3scale", "product", "export", "-k", "--file=/tmp/3scale/files/product.yaml", sourceInstance, sourceProductName])
  }

  stage("Edit product name") {
    runToolbox([ "sed", "-i", "s/$sourceProductName/$targetProductName/g", "/tmp/3scale/files/product.yaml" ])
  }

  stage("Edit application plan name") {
    runToolbox([ "sed", "-i", "s/$sourceAppPlanName/$targetAppPlanName/g", "/tmp/3scale/files/product.yaml" ])
  }

  stage("Edit backend name") {
    runToolbox([ "sed", "-i", "s/$sourceBackendName/$targetBackendName/g", "/tmp/3scale/files/product.yaml" ])
  }

  stage("Import product") {
    runToolbox([ "3scale", "product", "import", "-k", "--file=/tmp/3scale/files/product.yaml", targetInstance])
  }
  stage("Create an Application") {
    runToolbox([ "3scale", "application", "create", "-k", targetInstance, account, targetProductName, targetAppPlanName, targetApplicationName])
  }
}

/*
 * This function runs the 3scale toolbox as a Kubernetes Job.
 */
def runToolbox(args) {
  // You can adjust the Job Template to your needs
  def kubernetesJob = [
    "apiVersion": "batch/v1",
    "kind": "Job",
    "metadata": [
      "name": "toolbox"
    ],
    "spec": [
      // When the pipeline development is finished, you can increase the backoffLimit to 2
      "backoffLimit": 0,
      // Adjust the activeDeadlineSeconds according to your server velocity
      "activeDeadlineSeconds": 300,
      "template": [
        "spec": [
          "restartPolicy": "Never",
          "containers": [
            [
              "name": "job",
              // Could be changed to use local image in openshift registry after import
              "image": "quay.io/redhat/3scale-toolbox:master",
              "imagePullPolicy": "Always",
              "args": [ "3scale", "version" ],
              "env": [
                // This is needed for the 3scale_toolbox to read its configuration file
                // mounted from the toolbox-config secret 
                [ "name": "HOME", "value": "/tmp/3scale/config" ]
              ],
              "volumeMounts": [
                [ "mountPath": "/tmp/3scale/config", "name": "toolbox-config" ],
                [ "mountPath": "/tmp/3scale/files", "name": "3scale-volume" ]
              ]
            ]
          ],
          "volumes": [
            // This Secret contains the .3scalerc.yaml toolbox configuration file
            [ "name": "toolbox-config", "secret": [ "secretName": "3scalerc" ] ],
            //volume to put the yaml files to be used by different jobs 
            [ "name": "3scale-volume", "persistentVolumeClaim": [ "claimName": "3scale-pvc" ] ]
          ]
        ]
      ]
    ]
  ]
  
  // Patch the Kubernetes job template to add the provided 3scale_toolbox arguments
  kubernetesJob.spec.template.spec.containers[0].args = args

  // Write the Kubernetes Job definition to a YAML file
  sh "rm -f job.yaml"
  writeYaml file: "job.yaml", data: kubernetesJob


  // Do some cleanup, create the job and wait a wait for it to be completed
  sh """
  oc delete job toolbox --ignore-not-found
  sleep 2
  oc create -f job.yaml
  sleep 2
  oc wait --for=condition=complete job/toolbox
  """
  
  // ...before collecting logs!
  def logs = sh(script: "oc logs -f job/toolbox", returnStdout: true)
  
  // When using "returnStdout: true", Jenkins does not display stdout logs anymore.
  // So, we have to display them by ourselves!
  echo logs

  // The stdout logs may contains parseable output, so we return them to the caller
  // that will use them as desired.
  return logs
}
