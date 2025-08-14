pipeline {
  agent any
  options { timestamps(); ansiColor('xterm') }
  environment {
    REGISTRY = 'docker.io'
    NAMESPACE = 'learner-dev'
    FRONTEND_IMAGE = 'garapatimanoj/learner-fe'
    BACKEND_IMAGE = 'garapatimanoj/learner-be'
    FE_TAG = "dev-${env.BUILD_NUMBER}"
    BE_TAG = "dev-${env.BUILD_NUMBER}"
    CHART_PATH = 'deployment/charts/learner-report'
  }
  stages {
    stage('Checkout'){ steps{ checkout scm } }
    stage('Build & Push FE'){
      steps{
        dir('frontend'){
          script{
            docker.withRegistry("https://${env.REGISTRY}", 'dockerhub-creds'){
              def fe = docker.build("${FRONTEND_IMAGE}:${FE_TAG}")
              fe.push()
            }
          }
        }
      }
    }
    stage('Build & Push BE'){
      steps{
        dir('backend'){
          script{
            docker.withRegistry("https://${env.REGISTRY}", 'dockerhub-creds'){
              def be = docker.build("${BACKEND_IMAGE}:${BE_TAG}")
              be.push()
            }
          }
        }
      }
    }
    stage('Deploy (Helm)'){
      steps{
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]){
          sh '''
            export KUBECONFIG="$KUBECONFIG_FILE"
            kubectl get ns ${NAMESPACE} || kubectl create ns ${NAMESPACE}
            helm upgrade --install learner-report ${CHART_PATH} \
              --namespace ${NAMESPACE} \
              -f deployment/environments/values-dev.yaml \
              --set frontend.image=${FRONTEND_IMAGE},frontend.tag=${FE_TAG} \
              --set backend.image=${BACKEND_IMAGE},backend.tag=${BE_TAG}
          '''
        }
      }
    }
  }
}
