pipeline {

  agent { label 'busybox' }

  stages {

    stage('Checkout Source') {
      steps {
        git url:'https://github.com/eric-systex/playjenkins.git', branch:'test-deploy-stage'
      }
    }

    stage('Deploy App') {
      steps {
        script {
          kubernetesDeploy(configs: "nginx.yaml", kubeconfigId: "mykubeconfig")
        }
      }
    }

  }

}
