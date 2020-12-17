pipeline {

  agent { label 'kubepod' }

  stages {

    stage('Checkout Source') {
      steps {
        git url:'https://github.com/eric-systex/playjenkins.git', branch:'test-deploy-stage'
      }
    }

    stage('Deploy App') {
      steps {
        script {
          sh 'ls /home/jenkins/minikube'
          sh 'ls /home/jenkins/minikube/profiles/minikube'
        }
      }
    }

  }

}
