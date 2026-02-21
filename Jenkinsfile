pipeline {
  agent any

  tools { maven 'Maven-3.9.12' }

  environment {
    DOCKER_IMAGE = "demo-app"
    GITHUB_REPO  = "github.com/aaraya10371/java-mave-app.git"
    BRANCH_NAME  = "jenkins-jobs"
  }

  stages {
    stage('increment version') {
      steps {
        script {
          echo 'incrementing app version...'
          sh '''
            mvn build-helper:parse-version versions:set \
              -DnewVersion=\\${parsedVersion.majorVersion}.\\${parsedVersion.minorVersion}.\\${parsedVersion.nextIncrementalVersion} \
              versions:commit
          '''

          // ✅ Safe way (no ${project.version} interpolation by Jenkins)
          def version = sh(
            script: "mvn -q -DforceStdout help:evaluate -Dexpression=project.version",
            returnStdout: true
          ).trim()

          env.IMAGE_NAME = "${version}-${env.BUILD_NUMBER}"
          echo "Resolved version: ${version}"
          echo "IMAGE_NAME: ${env.IMAGE_NAME}"
        }
      }
    }

    stage('build app') {
      steps {
        echo 'building the application...'
        sh 'mvn clean package'
      }
    }

    stage('build image') {
      steps {
        script {
          echo 'building + pushing the docker image...'
          withCredentials([usernamePassword(credentialsId: 'docker-hub-repo', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
            sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
            sh "docker build -t ${DOCKER_USER}/${DOCKER_IMAGE}:${env.IMAGE_NAME} ."
            sh "docker push ${DOCKER_USER}/${DOCKER_IMAGE}:${env.IMAGE_NAME}"
          }
        }
      }
    }

    stage('deploy') {
      steps {
        echo 'deploying docker image... (placeholder)'
      }
    }

    stage('commit version update') {
      steps {
        script {
          echo 'committing version update back to GitHub...'
          withCredentials([usernamePassword(credentialsId: 'GitHub-Credentials', passwordVariable: 'GH_TOKEN', usernameVariable: 'GH_USER')]) {
            sh 'git config --global user.email "jenkins@example.com"'
            sh 'git config --global user.name "jenkins"'
            sh "git remote set-url origin https://${GH_USER}:${GH_TOKEN}@${GITHUB_REPO}"
            sh 'git add pom.xml'
            sh 'git commit -m "ci: version bump" || echo "No changes to commit"'
            sh "git push origin HEAD:${BRANCH_NAME}"
          }
        }
      }
    }
  }

  post {
    always { echo "Pipeline finished. Build: ${env.BUILD_NUMBER}" }
    failure { echo "Pipeline failed — check the stage logs above." }
  }
}
