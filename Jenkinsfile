#!groovy

/*
 * This pipeline will provision the Beer Catalog API found on the microcks/api-lifecycle
 * repository (secured using API Key) on a 3scale Hosted instance. 
 * 
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
def targetProductName = "hello_world_product_3"
def sourceAppPlanName = "hello_world_app_plan"
def targetAppPlanName = "hello_world_app_plan_3"
def sourceBackendName = "hello_world_backend"
def targetBackendName = "hello_world_backend_3"
def targetApplicationName = "hello_world_application_3"
def account = "hello_world_user"
/*
 * Only needed when using self-managed APIcast or on-premises installation of 3scale
 */
def publicStagingBaseURL = null // change to something such as "http://my-staging-api.example.test" for self-managed APIcast or on-premises installation of 3scale
def publicProductionBaseURL = null // change to something such as "http://my-production-api.example.test" for self-managed APIcast or on-premises installation of 3scale

node {
  // stage("Export product") {
  //   runToolbox([ "3scale", "product", "export", "-k", "--file=/tmp/3scale/files/product.yaml", sourceInstance, sourceProductName])
  // }

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
    runToolbox([ "3scale", "product", "import", "-k", "-f /tmp/3scale/files/product.yaml", targetInstance])
  }
  // stage("Create an Application") {
  //   runToolbox([ "3scale", "application", "create", "-k", targetInstance, account, targetProductName, targetAppPlanName, targetApplicationName])
  // }
}

  // stage("Run integration tests") {
  //   /*
  //    * When using 3scale Hosted with hosted APIcast instance, we need to extract the proxy definition
  //    * to read the Public Staging Base URL. Otherwise, we can just re-use the publicStagingBaseURL
  //    * variable defined above.
  //    */
  //   if (publicStagingBaseURL == null) {
  //     def proxyDefinition = runToolbox([ "3scale", "proxy", "show", targetInstance, targetSystemName, "sandbox" ])
  //     def proxy = readJSON text: proxyDefinition
  //     publicStagingBaseURL = proxy.content.proxy.sandbox_endpoint
  //   }

  //   sh """
  //   echo "Public Staging Base URL is ${publicStagingBaseURL}"
  //   echo "userkey is ${testUserKey}"
  //   curl -vfk ${publicStagingBaseURL}/beer -H 'api-key: ${testUserKey}'
  //   curl -vfk ${publicStagingBaseURL}/beer/Weissbier -H 'api-key: ${testUserKey}'
  //   curl -vfk ${publicStagingBaseURL}/beer/findByStatus/available -H 'api-key: ${testUserKey}'
  //   """
  // }
  
  // stage("Promote to production") {
  //   runToolbox([ "3scale", "proxy", "promote", targetInstance, targetSystemName ])
  // }

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
              // Do not forget to change the image tag to something more stable than "master"!
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
                // [ "mountPath": "/artifacts", "name": "artifacts" ],
                [ "mountPath": "/tmp/3scale/files", "name": "3scale-volume" ]
              ]
            ]
          ],
          "volumes": [
            // This Secret contains the .3scalerc.yaml toolbox configuration file
            [ "name": "toolbox-config", "secret": [ "secretName": "3scalerc" ] ],
            // This ConfigMap contains the artifacts to deploy (OpenAPI Specification file, Application Plan file, etc.)
            // [ "name": "artifacts", "configMap": [ "name": "openapi" ] ],
            [ "name": "3scale-volume", "persistentVolumeClaim": [ "claimName": "3scale-pvc" ] ]
          ]
        ]
      ]
    ]
  ]
  
  // def 3sacalerc = [
  //   ---
  //   :remotes:
  //     3scale-source
  //       :authentication: 
  //       :endpoint: 
  //     3scale-destination
  //       :authentication: 
  //       :endpoint: 
  // ]
  
  // Patch the Kubernetes job template to add the provided 3scale_toolbox arguments
  kubernetesJob.spec.template.spec.containers[0].args = args

  // Write the Kubernetes Job definition to a YAML file
  sh "rm -f job.yaml"
  writeYaml file: "job.yaml", data: kubernetesJob
  // writeYaml file: "3scalerc.yaml", data: kubernetesJob


  // Do some cleanup, create the job and wait a little bit...
  sh """
  oc delete job toolbox --ignore-not-found
  sleep 2
  oc create -f job.yaml
  i=0
  while [ true ]; do i=$[$i+1]; oc get job toolbox | grep "1/1"; if [ $? -eq 0 ]; then break; elif [ $i -eq 60 ]; then echo "ERROR"; break; fi; echo $i; sleep 1; done
  """
  // sh """
  // oc delete job toolbox --ignore-not-found
  // sleep 2
  // oc delete secret 3scalerc --ignore-not-found
  // sleep 2
  // oc create secret generic 3scalerc  --from-file=3scalerc.yaml
  // sleep 2
  // oc create -f job.yaml
  // sleep 20 # Adjust the sleep duration to your server velocity
  // """
  
  // ...before collecting logs!
  def logs = sh(script: "oc logs -f job/toolbox", returnStdout: true)
  
  // When using "returnStdout: true", Jenkins does not display stdout logs anymore.
  // So, we have to display them by ourselves!
  echo logs

  // The stdout logs may contains parseable output, so we return them to the caller
  // that will use them as desired.
  return logs
}
