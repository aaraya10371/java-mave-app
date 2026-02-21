pipeline {
    agent any

    tools {
        maven 'Maven-3.9.12'
    }

    environment {
        DOCKER_IMAGE = "demo-app"
        GITHUB_REPO_URL = "https://github.com/aaraya10371/java-mave-app.git"
        GIT_BRANCH = "jenkins-jobs"
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

                    // Get the project version safely (no Groovy ${project.version} interpolation)
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
                // Add deploy commands later (k8s / docker run / etc.)
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

                        sh 'git config --global user.email "jenkins@example.com"'
                        sh 'git config --global user.name "jenkins"'

                        sh 'git status'
                        sh 'git add pom.xml'
                        sh 'git commit -m "ci: version bump" || echo "No changes to commit"'

                        // Robust push: don't put token in URL (avoids special-char URL issues)
                        sh '''
                            git -c http.extraHeader="Authorization: Basic $(echo -n $GH_USER:$GH_TOKEN | base64)" \
                            push ${GITHUB_REPO_URL} HEAD:${GIT_BRANCH}
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
