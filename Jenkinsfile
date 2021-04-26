def SERNAME='unknown'
pipeline {
    agent { node { label 'docker' } }
    environment {
       COMPOSE_PROJECT_NAME = "${env.JOB_NAME}_${env.BUILD_ID}"
    }
    stages {
        stage('Prepare') {
            steps {
                sh 'whoami'
                sh '''
                    docker info
                    pwd
                    env | sort
                   '''
            }
        }
        stage('Docker build and start') {
            steps {
                sh "echo $JOB_NAME"
                sh 'echo $BUILD_ID'
                sh "echo $COMPOSE_PROJECT_NAME"
            }
        }
        stage('Compile') {
            steps {
              script {
                 configFileProvider([
                     configFile(
                         fileId: 'c2b74139-8c8f-4ba3-be52-47a692f92206',
                         targetLocation: '.',
                         variable: 'MVN_NEXUS')
                 ]) {}

              }
              sh 'docker run --user $(id -u):$(id -g) -i --rm --name jsoup-${BUILD_NUMBER} -v "$PWD":/usr/src/mymaven -v "$HOME/.m2":/usr/src/mymaven/.m2 -w /usr/src/mymaven -e MAVEN_CONFIG=/usr/src/mymaven maven:3.6-jdk-11 mvn -s nexus_maven_settings_xml compile -U'
            }
        }
        stage('Deploy artifacts and archetype master') {
            steps {
              script {
                configFileProvider([
                    configFile(
                        fileId: 'c2b74139-8c8f-4ba3-be52-47a692f92206',
                        targetLocation: '.',
                        variable: 'MVN_NEXUS')
                ]) {}
              }
             sh 'docker run --user $(id -u):$(id -g) -i --rm --name jsoup-${BUILD_NUMBER} -v "$PWD":/usr/src/mymaven -v "$HOME/.m2":/usr/src/mymaven/.m2 -w /usr/src/mymaven -e MAVEN_CONFIG=/usr/src/mymaven maven:3.6-jdk-11 mvn -s nexus_maven_settings_xml deploy'
            }
        }
        stage('Rebuild Nexus master metadata') {
            steps {
                withCredentials([usernamePassword(
                        credentialsId: 'NEXUS_ADMIN',
                        usernameVariable: 'NEXUS_ADMIN_USER',
                        passwordVariable: 'NEXUS_ADMIN_PWD')
                ]) {
                    sh 'curl -v -L --user ${NEXUS_ADMIN_USER}:${NEXUS_ADMIN_PWD} -X POST "https://repositories-proxy.mrf.io/nexus/service/rest/v1/tasks/c36675fa-fe6e-42a3-9f54-68f13f87a999/run" '
                }
            }
        }
        stage('Deploy artifacts and archetype secondary') {
            steps {
              script {
                configFileProvider([
                    configFile(
                        fileId: 'c2b74139-8c8f-4ba3-be52-47a692f92206',
                        targetLocation: '.',
                        variable: 'MVN_NEXUS')
                ]) {}
              }
              sh 'docker run --user $(id -u):$(id -g) -i --rm --name jsoup-${BUILD_NUMBER} -v "$PWD":/usr/src/mymaven -v "$HOME/.m2":/usr/src/mymaven/.m2 -w /usr/src/mymaven -e MAVEN_CONFIG=/usr/src/mymaven maven:3.6-jdk-11 mvn -s nexus_maven_settings_xml deploy -P secondary,!master -Dmaven.test.skip=true -DskipTests'
            }
        }
        stage('Rebuild Nexus secondary metadata') {
            steps {
                withCredentials([usernamePassword(
                        credentialsId: 'NEXUS_ADMIN',
                        usernameVariable: 'NEXUS_ADMIN_USER',
                        passwordVariable: 'NEXUS_ADMIN_PWD')
                ]) {
                    sh 'curl -v -L --user ${NEXUS_ADMIN_USER}:${NEXUS_ADMIN_PWD} -X POST "https://repositories.mrf.io/nexus/service/rest/v1/tasks/3f581f66-3222-4c41-87d6-85604d6963a3/run" '
                }
            }
        }
    }
    post {
      always {
          sh 'echo "Job ran @ ${JENKINS_NODE_NAME}"'
      }
      success {
          sh 'sleep 30'
      }
      failure {
          slackSend channel: '#mfeels',
             color: '#ff0000',
             message: "The pipeline ${currentBuild.fullDisplayName} has failed."
      }
    }
  options {
    buildDiscarder logRotator(artifactDaysToKeepStr: '60', artifactNumToKeepStr: '60', daysToKeepStr: '60', numToKeepStr: '60')
    timeout(time: 20, unit: 'MINUTES')
    timestamps()
    disableConcurrentBuilds()
  }
}
