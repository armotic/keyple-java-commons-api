#!groovy
def keypleVersion
pipeline {
  environment {
    PROJECT_NAME = "keyple-java-commons-api"
    PROJECT_BOT_NAME = "Eclipse Keyple Bot"
  }
  agent {
    kubernetes {
      yaml javaBuilder('2.0')
    }
  }
  stages {
    stage('Import keyring') {
      when {
        expression { env.GIT_URL.startsWith('https://github.com/armotic/keyple-') && env.CHANGE_ID == null }
      }
      steps {
        container('java-builder') {
          withCredentials([file(credentialsId: 'secret-subkeys.asc', variable: 'KEYRING')]) {
            sh '/usr/local/bin/import_gpg "${KEYRING}"'
          }
        }
      }
    }
    stage('Prepare settings') {
      steps {
        container('java-builder') {
          sh 'printenv'
          script {
            keypleVersion = sh(script: 'grep version gradle.properties | cut -d= -f2 | tr -d "[:space:]"', returnStdout: true).trim()
            echo "Building version ${keypleVersion} in branch ${env.GIT_BRANCH}"
            deploySnapshot = env.GIT_URL == "https://github.com/eclipse/${env.PROJECT_NAME}.git" && env.GIT_BRANCH == "main" && env.CHANGE_ID == null
            deployRelease = env.GIT_URL == "https://github.com/eclipse/${env.PROJECT_NAME}.git" && (env.GIT_BRANCH == "master" || env.GIT_BRANCH.startsWith('release-')) && env.CHANGE_ID == null && keypleVersion ==~ /\d+\.\d+.\d+$/
          }
        }
      }
    }
    stage('Keyple Java: Build and Test') {
      when {
        expression { !deploySnapshot && !deployRelease }
      }
      parallel {
        stage('Keyple Java 6: Build and Test') {
          steps {
            container('java-builder') {
              sh './gradlew clean assemble -P javaSourceLevel=1.6 -P javaTargetLevel=1.6 -P buildDir=build-6 --no-build-cache --info'
              catchError(buildResult: 'UNSTABLE', message: 'There were failing tests.', stageResult: 'UNSTABLE') {
                sh './gradlew test -P javaSourceLevel=1.6 -P javaTargetLevel=1.6 -P buildDir=build-6 --no-build-cache --info'
              }
              junit allowEmptyResults: true, testResults: 'build-+/test-results/test/*.xml'
            }
          }
        }
        stage('Keyple Java 8: Build and Test') {
          steps {
            container('java-builder') {
              sh './gradlew clean assemble -P javaSourceLevel=1.8 -P javaTargetLevel=1.8 -P buildDir=build-8 --no-build-cache --info'
              catchError(buildResult: 'UNSTABLE', message: 'There were failing tests.', stageResult: 'UNSTABLE') {
                sh './gradlew test -P javaSourceLevel=1.8 -P javaTargetLevel=1.8 -P buildDir=build-8 --no-build-cache --info'
              }
              junit allowEmptyResults: true, testResults: 'build-8/test-results/test/*.xml'
            }
          }
        }
        stage('Keyple Java 11: Build and Test') {
          steps {
            container('java-builder') {
              sh './gradlew clean assemble -P javaSourceLevel=11 -P javaTargetLevel=11 -P buildDir=build-11 --no-build-cache --info'
              catchError(buildResult: 'UNSTABLE', message: 'There were failing tests.', stageResult: 'UNSTABLE') {
                sh './gradlew test -P javaSourceLevel=11 -P javaTargetLevel=11 -P buildDir=build-11 --no-build-cache --info'
              }
              junit allowEmptyResults: true, testResults: 'build-11/test-results/test/*.xml'
            }
          }
        }
      }
    }
    stage('Keyple Java: Build, Test and Publish') {
      when {
        expression { deploySnapshot }
      }
      steps {
        container('java-builder') {
          configFileProvider([configFile(fileId: 'gradle.properties',
            targetLocation: '/home/jenkins/agent/gradle.properties')]) {
            sh './gradlew clean build test publish --info'
            sh './gradlew --stop'
            junit allowEmptyResults: true, testResults: 'build/test-results/test/*.xml'
          }
        }
      }
    }
    stage('Keyple Java: Release') {
      when {
        expression { deployRelease }
      }
      steps {
        container('java-builder') {
          configFileProvider([configFile(fileId: 'gradle.properties',
            targetLocation: '/home/jenkins/agent/gradle.properties')]) {
            sh './gradlew clean release --info'
            sh './gradlew --stop'
            junit allowEmptyResults: true, testResults: 'build/test-results/test/*.xml'
          }
        }
      }
    }
    stage('Keyple Java: Code Quality') {
      when {
        expression { deploySnapshot || deployRelease }
      }
      steps {
        catchError(buildResult: 'SUCCESS', message: 'Unable to log code quality to Sonar.', stageResult: 'FAILURE') {
          container('java-builder') {
            withCredentials([string(credentialsId: 'sonarcloud-token', variable: 'SONAR_LOGIN')]) {
              sh './gradlew sonarqube --info'
              sh './gradlew --stop'
            }
          }
        }
      }
    }
    stage('Keyple Java: Publish packaging to Eclipse') {
      when {
        expression { deploySnapshot || deployRelease }
      }
      steps {
        container('java-builder') {
            sh 'publish_packaging'
        }
      }
    }
  }
  post {
    always {
      container('java-builder') {
        archiveArtifacts artifacts: 'build*/reports/tests/**', allowEmptyArchive: true
      }
    }
  }
}
