pipeline {
  agent any
  stages {
    stage('install dependencies') {
      post {
        always {
          sh 'bash ./bash-scripts/clearDockerImages.sh'
        }

      }
      steps {
        dir(path: 'node-app') {
          sh 'npm install'
        }

      }
    }

    stage(' Build and Test') {
      parallel {
        stage('Test') {
          steps {
            dir(path: 'node-app') {
              sh 'npm run  test:unit'
            }

          }
        }

        stage('Build') {
          steps {
            dir(path: 'node-app') {
              sh 'npm run build'
            }

          }
        }

      }
    }

    stage('Build Docker Image') {
      post {
        failure {
          sh 'bash ./bash-scripts/clearDockerImages.sh'
        }

      }
      steps {
        script {
          dir("node-app") {

            dockerImage = docker.build registry + ":$BUILD_NUMBER"
          }
        }

      }
    }

    stage('push image to docker hup') {
      steps {
        script {
          docker.withRegistry('', registryCredential) {
            dockerImage.push()
          }
        }

      }
    }

    stage('Update K8s Green deployment with new image ') {
      steps {
        sh """
                sed -i 's|DOCKER|$registry:$BUILD_NUMBER|g' ./k8s/green-deployment.yaml
              """
      }
    }

    stage('create kubecontext file') {
      steps {
        withCredentials(bindings: [aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
          sh 'aws eks update-kubeconfig --region us-east-2 --name jenkins-cluster '
          sh 'kubectl get nodes'
        }

      }
    }

    stage('Make sure  that we have geen-namespace and blue-ns ') {
      steps {
        withCredentials(bindings: [aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
          sh 'bash ./bash-scripts/cheackForNameSpaces.sh'
        }

      }
    }

    stage('Deploy blue deployment if not exsit ') {
      steps {
        withCredentials(bindings: [aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
          sh 'bash  ./bash-scripts/check-if-blue-is-exist-ordeploy-it-if-not.sh'
        }

      }
    }

    stage('Deploy Green deployment if not exsit ') {
      steps {
        withCredentials(bindings: [aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
          sh '  kubectl  apply  -f ./k8s/green-deployment.yaml '
          sh '  kubectl  apply  -f ./k8s/green-service.yaml '
        }

      }
    }

    stage('Smoke Test') {
      steps {
        withCredentials(bindings: [aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
          sh 'sleep 10'
          sh ' kubectl get service,pod --namespace=green-deployment --all-namespaces=true'
          sh ' bash ./bash-scripts/CheckForPodsGetReady.sh'
          sh 'bash ./bash-scripts/smokeTest.sh'
        }

      }
    }

    stage('hey we just deploy green version ') {
      steps {
        input 'we just deploy green version after approving we going to update production'
      }
    }

    stage('update blue app with new docker Image ') {
      steps {
        sh """
                sed -i 's|hossamalsankary/nodejs_app:49|$registry:$BUILD_NUMBER|g' ./k8s/green-deployment.yaml

                """
        withCredentials(bindings: [aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
          sh '  kubectl  apply  -f ./k8s/blue-deployment.yaml '
          sh '  kubectl  apply  -f ./k8s/blue-service.yaml '
        }

      }
    }

    stage('Destroy Green version') {
      steps {
        withCredentials(bindings: [aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
          sh 'bash ./bash-scripts/clear-green-deployment.sh'
        }

      }
    }

  }
  environment {
    registry = 'hossamalsankary/nodejs_app'
    registryCredential = 'docker_credentials'
    ANSIBLE_PRIVATE_KEY = credentials('secritfile')
  }
  post {
    failure {
      sh 'bash ./bash-scripts/clearDockerImages.sh'
    }

  }
}