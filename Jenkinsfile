#!/usr/bin/groovy

// Load shared libraries from github jenkins-shared-libraries
@Library('shared-library') _

def appName = "sandbox"
def microName = "maven"
def agentLabel = "maven"
def targetEnv = "dev"
def cicdProject = "jenkins"
def devProject = appName + "-" + targetEnv
def imageName = appName + "-" + microName 
def commitRef = ref.split('/')
def releaseCommit = (commitRef[1] == "tags" ? true : false)

pipeline {
  // If not specified because it's not relevant, start up a 'nodejs' agent by default
  agent { label agentLabel.trim() ?: "nodejs" }

  environment {
    gitURL = "https://github.com/jclaret/gitops-deploy.git"
  }

  options {
    ansiColor('xterm')
    buildDiscarder(logRotator(numToKeepStr: '5'))
    timeout(time: 30, unit: 'MINUTES')
    skipDefaultCheckout(true)
  }

  stages {
    stage ('Checkout SCM') {
      steps {
        checkout scm: [$class: 'GitSCM', userRemoteConfigs: [[url: gitURL , credentialsId: gitKey ]], 
                branches: [[name: ref]]], 
                poll: false
        sh 'echo "$ref" > ref.txt'
      }
    }
    input 'stop'
    stage('Build Image') {
      steps {
        script {
          // set environment variables to dev
          setConfiguration(appName, devProject, imageName)
          openshift.withCluster() {
            openshift.withProject(cicdProject) {
              def sb = openshift.selector("bc", imageName)
                  .startBuild("--from-dir=.","--wait=true","--follow=true", "--build-loglevel=5")
            }
          }
        }
      }
    }
    stage('Deploy') {
      steps {
        script {
          deployImage(imageName, devProject)            
        }
      }
    }
    stage('Push to Quay') {
      when {
        beforeAgent true
        expression { releaseCommit }
      }
      agent { label 'skopeo' }
      steps {
        script {
          def versionTag = commitRef[2]
          pushImageToQuay(devProject, imageName, versionTag)
        }
      }
    }
  }

  post {
    always {
      // Clean up the workspace
      cleanWs()
    }
  }
}
