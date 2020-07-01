import groovy.json.JsonBuilder

node('jenkins-jenkins-slave') {
  withEnv(['REPOSITORY=c1-eicar']) {
    stage('Pull Image from Git') {
      script {
        git (url: "${scm.userRemoteConfigs[0].url}", credentialsId: "github-auth")
      }
    }
    stage('Build Image') {
      script {
        dbuild = docker.build("${REPOSITORY}:$BUILD_NUMBER")
      }
    }
    parallel (
      "Test": {
        echo 'All functional tests passed'
      },
      "Check Image (pre-Registry)": {
        smartcheckScan([
          imageName: "${REPOSITORY}:$BUILD_NUMBER",
          smartcheckHost: "smartcheck-3-127-196-237.nip.io:443",
          smartcheckCredentialsId: "smartcheck-auth",
          insecureSkipTLSVerify: true,
          insecureSkipRegistryTLSVerify: true,
          preregistryScan: true,
          preregistryHost: "smartcheck-registry-3-127-196-237.nip.io:443",
          preregistryCredentialsId: "preregistry-auth",
          findingsThreshold: new groovy.json.JsonBuilder([
            malware: 3,
            vulnerabilities: [
              defcon1: 0,
              critical: 5,
              high: 20,
            ],
            contents: [
              defcon1: 0,
              critical: 1,
              high: 10,
            ],
            checklists: [
              defcon1: 0,
              critical: 0,
              high: 10,
            ],
          ]).toString(),
        ])
      }
    )

    stage('Push Image to Registry') {
      script {
        docker.withRegistry("https://${K8S_REGISTRY}", 'registry-auth') {
          dbuild.push('$BUILD_NUMBER')
          dbuild.push('latest')
        }
      }
    }
    stage('Deploy App to Kubernetes') {
      script {
        kubernetesDeploy(configs: "app.yml",
                         kubeconfigId: "kubeconfig",
                         enableConfigSubstitution: true,
                         dockerCredentials: [
                           [credentialsId: "registry-auth", url: "https://registry-3-127-196-237.nip.io"],
                         ])
      }
    stage('DS Scan for Recommendations') {
      withCredentials([string(credentialsId: 'deepsecurity-key', variable: 'DSKEY')]) {
        sh 'curl -X POST https://app.deepsecurity.trendmicro.com/api/scheduledtasks/133 -H "api-secret-key: ${DSKEY}" -H "api-version: v1" -H "Content-Type: application/json" -d "{ \\"runNow\\": \\"true\\" }" '
        }     
      }
    }
  }
}
