pipeline {
    agent any

    tools {
        maven 'Maven-3.9.12'
    }

    environment {
        DOCKER_IMAGE    = "demo-app"
        GITHUB_REPO_URL = "https://github.com/aaraya10371/java-mave-app.git"
        GIT_BRANCH      = "jenkins-jobs"
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

                    // Safe way to read project.version
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

                    withCredentials([usernamePassword(
                        credentialsId: 'docker-hub-repo',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        // Avoid leaking secrets in logs
                        sh '''
                            set +x
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                            set -x
                        '''

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

                    withCredentials([usernamePassword(
                        credentialsId: 'GitHub-Credentials',
                        usernameVariable: 'GH_USER',
                        passwordVariable: 'GH_TOKEN'
                    )]) {

                        sh '''
                            set +x
                            git config --global user.email "jenkins@example.com"
                            git config --global user.name "jenkins"

                            git add pom.xml
                            git commit -m "ci: version bump" || echo "No changes to commit"

                            # Create an askpass helper so git can authenticate non-interactively
                            cat > /tmp/git_askpass.sh <<'EOF'
#!/bin/sh
case "$1" in
  Username*) echo "$GH_USER" ;;
  Password*) echo "$GH_TOKEN" ;;
  *) echo "" ;;
esac
EOF
                            chmod +x /tmp/git_askpass.sh

                            export GIT_ASKPASS=/tmp/git_askpass.sh
                            export GIT_TERMINAL_PROMPT=0

                            # Push without embedding token in URL
                            git push ${GITHUB_REPO_URL} HEAD:${GIT_BRANCH}
                            set -x
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished. Build: ${env.BUILD_NUMBER}"
        }
        failure {
            echo "Pipeline failed — check the stage logs above."
        }
    }
}
