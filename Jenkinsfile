#!groovy

// Run this pipeline on the custom Maven Slave ('maven')
// Maven Slaves have JDK and Maven installed.

def app_git_repo       = 'https://github.com/elos-tech/openshift-cicd-app.git'
def name_prefix        = 'cicd'
def components_project = "${name_prefix}-components"
def app_project_dev    = "${name_prefix}-tasks-dev"
def app_project_prod   = "${name_prefix}-tasks-prod"

podTemplate(
  label: "maven-pod",
  cloud: "openshift",
  containers: [
    containerTemplate(
      name: "jnlp",
      workingDir: '/tmp',
      image: "docker-registry.default.svc:5000/${name_prefix}-jenkins/jenkins-agent-appdev"
    )
  ]
) {
  node('maven-pod') {

    // Define Maven Command. Make sure it points to the correct
    // settings for our Nexus installation (use the service to
    // bypass the router). The file nexus_openshift_settings.xml
    // needs to be in the Source Code repository.
    def mvnCmd = "mvn -s ./nexus_openshift_settings.xml"

    // Checkout Source Code
    stage('Checkout Source') {
      git app_git_repo
    }

    // The following variables need to be defined at the top level
    // and not inside the scope of a stage - otherwise they would not
    // be accessible from other stages.
    // Extract version and other properties from the pom.xml
    def groupId    = getGroupIdFromPom("pom.xml")
    def artifactId = getArtifactIdFromPom("pom.xml")
    def version    = getVersionFromPom("pom.xml")
  
    // Set the tag for the development image: version + build number
    def devTag  = "${version}-${BUILD_NUMBER}"
    // Set the tag for the production image: version
    def prodTag = "${version}"
  
    // Using Maven build the war file
    // Do not run tests in this step
    stage('Build war') {
      echo "Building version ${version}"
    
      sh "${mvnCmd} clean package -DskipTests"
    }
  
    // Using Maven run the unit tests
    stage('Unit Tests') {
      echo "Running Unit Tests"
      sh "${mvnCmd} test"
    }
  
    // Using Maven call SonarQube for Code Analysis
    stage('Code Analysis') {
      echo "Running Code Analysis"
    
      sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube.${components_project}.svc:9000 -Dsonar.projectName=${JOB_BASE_NAME} -Dsonar.projectVersion=${devTag}"
    }
  
    // Publish the built war file to Nexus
    stage('Publish to Nexus') {
      echo "Publish to Nexus"
    
      sh "${mvnCmd} deploy -DskipTests=true -DaltDeploymentRepository=nexus::default::http://nexus3.${component_project}.svc:8081/repository/releases"
    }
  
    // Build the OpenShift Image in OpenShift and tag it.
    stage('Build and Tag OpenShift Image') {
      echo "Building OpenShift container image tasks:${devTag}"
   
      // Use the file you just published into Nexus:
      sh "oc start-build tasks --follow --from-file=http://nexus3.${component_project}.svc:8081/repository/releases/org/jboss/quickstarts/eap/tasks/${version}/tasks-${version}.war -n ${app_project_dev}"
  
      // Tag the image using the devTag
      openshiftTag alias: 'false', destStream: 'tasks', destTag: devTag, destinationNamespace: "$app_project_dev", namespace: "$app_project_dev", srcStream: 'tasks', srcTag: 'latest', verbose: 'false'
    }
  
    // Copy Image to Nexus Docker Registry
    stage('Copy Image to Nexus Docker Registry') {
      echo "Copy image to Nexus Docker Registry"
  
      sh "skopeo copy --src-tls-verify=false --dest-tls-verify=false --src-creds openshift:\$(oc whoami -t) --dest-creds admin:admin123 docker://docker-registry.default.svc.cluster.local:5000/${app_project_dev}/tasks:${devTag} docker://nexus-registry.${components_project}.svc.cluster.local:5000/tasks:${devTag}"
    
      openshiftTag alias: 'false', destStream: 'tasks', destTag: prodTag, destinationNamespace: "$app_project_dev", namespace: "$app_project_dev", srcStream: 'tasks', srcTag: devTag, verbose: 'false'
    }
  
    // Deploy the built image to the Development Environment.
    stage('Deploy to Dev') {
      echo "Deploying container image to Development Project"
      // Update the Image on the Development Deployment Config
      sh "oc set image dc/tasks tasks=docker-registry.default.svc:5000/${app_project_dev}/tasks:${devTag} -n ${app_project_dev}"
  
      // Update the Config Map which contains the users for the Tasks application
      sh "oc delete configmap tasks-config -n $app_project_dev --ignore-not-found=true"
      sh "oc create configmap tasks-config --from-file=./configuration/application-users.properties --from-file=./configuration/application-roles.properties -n ${app_project_dev}"
  
      // Deploy the development application.
      openshiftDeploy depCfg: 'tasks', namespace: "$app_project_dev", verbose: 'false', waitTime: '', waitUnit: 'sec'
      openshiftVerifyDeployment depCfg: 'tasks', namespace: "$app_project_dev", replicaCount: '1', verbose: 'false', verifyReplicaCount: 'false', waitTime: '', waitUnit: 'sec'
      openshiftVerifyService namespace: "$app_project_dev", svcName: 'tasks', verbose: 'false'
    }
  
    // Run Integration Tests in the Development Environment.
    stage('Integration Tests') {
      echo "Running Integration Tests"
      sleep 30
  
      // Create a new task called "integration_test_1"
      echo "Creating task"
      sh "curl -i -f -u 'tasks:redhat1' -H 'Content-Length: 0' -X POST http://tasks.${app_project_dev}.svc:8080/ws/tasks/integration_test_1"
  
      // Retrieve task with id "1"
      echo "Retrieving tasks"
      sh "curl -i -f -u 'tasks:redhat1' -H 'Content-Length: 0' -X GET http://tasks.${app_project_dev}.svc:8080/ws/tasks/1"
  
      // Delete task with id "1"
      echo "Deleting tasks"
      sh "curl -i -f -u 'tasks:redhat1' -H 'Content-Length: 0' -X DELETE http://tasks.${app_project_dev}.svc:8080/ws/tasks/1"
    }

    // Blue/Green Deployment into Production
    // Do not activate the new version yet.
    def destApp   = "tasks-green"
    def activeApp = ""
   
    stage('Blue/Green Production Deployment') {
      // Replace xyz-tasks-dev and xyz-tasks-prod with
      // your project names
      activeApp = sh(returnStdout: true, script: "oc get route tasks -n ${app_project_prod} -o jsonpath='{ .spec.to.name }'").trim()
    
      if (activeApp == "tasks-green") {
        destApp = "tasks-blue"
      }
    
      echo "Active Application:      " + activeApp
      echo "Destination Application: " + destApp
    
      // Update the Image on the Production Deployment Config
      sh "oc set image dc/${destApp} ${destApp}=docker-registry.default.svc:5000/${app_project_dev}/tasks:${prodTag} -n ${app_project_prod}"
    
      // Update the Config Map which contains the users for the Tasks application
      sh "oc delete configmap ${destApp}-config -n ${app_project_prod} --ignore-not-found=true"
      sh "oc create configmap ${destApp}-config --from-file=./configuration/application-users.properties --from-file=./configuration/application-roles.properties -n ${app_project_prod}"
    
      // Deploy the inactive application.
      // Replace xyz-tasks-prod with the name of your production project
      openshiftDeploy depCfg: destApp, namespace: "${app_project_prod}", verbose: 'false', waitTime: '', waitUnit: 'sec'
      openshiftVerifyDeployment depCfg: destApp, namespace: "${app_project_prod}", replicaCount: '1', verbose: 'false', verifyReplicaCount: 'true', waitTime: '', waitUnit: 'sec'
      openshiftVerifyService namespace: "${app_project_prod}", svcName: destApp, verbose: 'false'
    }
    
    // Switch stage (user input):
    stage('Switch over to new Version') {
      input "Switch Production?"
    
      echo "Switching Production application to ${destApp}."
    
      // Replace xyz-tasks-prod with the name of your production project
      sh 'oc patch route tasks -n ' + app_project_prod + ' -p \'{"spec":{"to":{"name":"' + destApp + '"}}}\''
    }
  }
}

// Convenience Functions to read variables from the pom.xml
// Do not change anything below this line.
// --------------------------------------------------------
def getVersionFromPom(pom) {
  def matcher = readFile(pom) =~ '<version>(.+)</version>'
  matcher ? matcher[0][1] : null
}

def getGroupIdFromPom(pom) {
  def matcher = readFile(pom) =~ '<groupId>(.+)</groupId>'
  matcher ? matcher[0][1] : null
}

def getArtifactIdFromPom(pom) {
  def matcher = readFile(pom) =~ '<artifactId>(.+)</artifactId>'
  matcher ? matcher[0][1] : null
}

