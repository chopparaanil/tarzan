
pipeline {
    agent { label 'linux-docker' }

    triggers {
        githubPush()
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '20'))
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
        timestamps()
        timeout(time: 45, unit: 'MINUTES')
    }

    parameters {
        string(
            name: 'MANUAL_SOURCE_BRANCH',
            defaultValue: 'master',
            description: 'Branch to check out for manual builds.'
        )

        booleanParam(
            name: 'FORCE_ARGOCD_SYNC',
            defaultValue: false,
            description: 'Ask Argo CD to sync immediately after the GitOps commit.'
        )
    }

    environment {
        // Application repository (GitHub)
        DEPLOY_BRANCH = 'master'
        APPLICATION_REPOSITORY = 'https://github.com/chopparaanil/tarzan.git'
        APPLICATION_GIT_CREDENTIALS_ID = 'github-app-read'

        // Harbor container registry
        HARBOR_HOST = 'harbor.example.com'
        HARBOR_PROJECT = 'tarzan'
        HARBOR_CREDENTIALS_ID = 'harbor-robot-push'

        API_IMAGE = "${HARBOR_HOST}/${HARBOR_PROJECT}/tarzan-api"
        WEB_IMAGE = "${HARBOR_HOST}/${HARBOR_PROJECT}/tarzan-web"

        // SonarQube
        SONARQUBE_ENV = 'sonarqube'
        SONAR_SCANNER_TOOL = 'SonarScanner'

        // GitOps repository
        GITOPS_REPOSITORY = 'ssh://git@gitlab.example.com/platform/tarzan-gitops.git'
        GITOPS_BRANCH = 'master'
        GITOPS_VALUES_FILE = 'environments/production/tarzan-values.yaml'
        GITOPS_CREDENTIALS_ID = 'gitlab-gitops-ssh'

        // Argo CD
        ARGOCD_SERVER = 'argocd.example.com'
        ARGOCD_APP = 'tarzan'
        ARGOCD_TOKEN_CREDENTIALS_ID = 'argocd-jenkins-token'
    }

    stages {

        stage('Checkout') {
            steps {
                script {
                    def sourceBranch = params.MANUAL_SOURCE_BRANCH?.trim()

                    if (!sourceBranch ||
                        !(sourceBranch ==~ /[A-Za-z0-9._/-]+/)) {
                        error("Invalid Git branch name: ${sourceBranch}")
                    }

                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/${sourceBranch}"]],
                        extensions: [
                            [$class: 'CloneOption',
                             noTags: false,
                             shallow: false]
                        ],
                        userRemoteConfigs: [[
                            credentialsId: env.APPLICATION_GIT_CREDENTIALS_ID,
                            url: env.APPLICATION_REPOSITORY
                        ]]
                    ])

                    env.IMAGE_TAG = sh(
                        label: 'Calculate immutable image tag',
                        returnStdout: true,
                        script: 'git rev-parse --short=12 HEAD'
                    ).trim()

                    env.SOURCE_BRANCH = sourceBranch
                    env.PUBLISH_RELEASE =
                        (sourceBranch == env.DEPLOY_BRANCH).toString()

                    currentBuild.description =
                        "${sourceBranch} @ ${env.IMAGE_TAG}"
                }

                sh '''
                    set -eu
                    echo "Source branch: $SOURCE_BRANCH"
                    echo "Image tag: $IMAGE_TAG"
                    echo "Publish release: $PUBLISH_RELEASE"
                '''
            }
        }

        stage('Agent prerequisites') {
            steps {
                sh '''
                    set -eu
                    node --version
                    npm --version
                    git --version
                    docker version
                    yq --version
                '''
            }
        }

        stage('Install dependencies') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Quality checks') {
            steps {
                sh '''
                    set -eu
                    npm run format:check
                    npm run lint
                    npm run typecheck
                    npm test
                    npm run build
                '''
            }
        }

        stage('SonarQube analysis') {
            steps {
                script {
                    def scannerHome = tool env.SONAR_SCANNER_TOOL

                    withSonarQubeEnv(env.SONARQUBE_ENV) {
                        withEnv(["SONAR_SCANNER_HOME=${scannerHome}"]) {
                            sh '''
                                set -eu
                                "$SONAR_SCANNER_HOME/bin/sonar-scanner" \
                                    -Dsonar.projectVersion="$IMAGE_TAG"
                            '''
                        }
                    }
                }
            }
        }

        stage('SonarQube quality gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build container images') {
            when {
                environment name: 'PUBLISH_RELEASE', value: 'true'
            }
            steps {
                sh '''
                    set -eu

                    docker build \
                        --file docker/api.Dockerfile \
                        --tag "$API_IMAGE:$IMAGE_TAG" \
                        .

                    docker build \
                        --file docker/web.Dockerfile \
                        --build-arg VITE_API_URL=/api \
                        --tag "$WEB_IMAGE:$IMAGE_TAG" \
                        .
                '''
            }
        }

        stage('Push images to Harbor') {
            when {
                environment name: 'PUBLISH_RELEASE', value: 'true'
            }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: env.HARBOR_CREDENTIALS_ID,
                        usernameVariable: 'HARBOR_USER',
                        passwordVariable: 'HARBOR_PASSWORD'
                    )
                ]) {
                    sh '''
                        set -eu
                        set +x

                        printf '%s' "$HARBOR_PASSWORD" |
                            docker login "$HARBOR_HOST" \
                                --username "$HARBOR_USER" \
                                --password-stdin

                        docker push "$API_IMAGE:$IMAGE_TAG"
                        docker push "$WEB_IMAGE:$IMAGE_TAG"

                        docker logout "$HARBOR_HOST"
                    '''
                }
            }
        }

        stage('Update GitOps image tags') {
            when {
                environment name: 'PUBLISH_RELEASE', value: 'true'
            }
            steps {
                dir('gitops-repository') {
                    deleteDir()

                    withCredentials([
                        sshUserPrivateKey(
                            credentialsId: env.GITOPS_CREDENTIALS_ID,
                            keyFileVariable: 'GITOPS_SSH_KEY',
                            usernameVariable: 'GITOPS_SSH_USER'
                        )
                    ]) {
                        sh '''
                            set -eu

                            export GIT_SSH_COMMAND="ssh -i $GITOPS_SSH_KEY -o IdentitiesOnly=yes"

                            git clone \
                                --branch "$GITOPS_BRANCH" \
                                --single-branch \
                                "$GITOPS_REPOSITORY" .

                            test -f "$GITOPS_VALUES_FILE"

                            export API_IMAGE WEB_IMAGE IMAGE_TAG

                            yq eval -i \
                                '.api.image.repository = strenv(API_IMAGE)' \
                                "$GITOPS_VALUES_FILE"

                            yq eval -i \
                                '.api.image.tag = strenv(IMAGE_TAG)' \
                                "$GITOPS_VALUES_FILE"

                            yq eval -i \
                                '.web.image.repository = strenv(WEB_IMAGE)' \
                                "$GITOPS_VALUES_FILE"

                            yq eval -i \
                                '.web.image.tag = strenv(IMAGE_TAG)' \
                                "$GITOPS_VALUES_FILE"

                            git config user.name 'Jenkins Tarzan Pipeline'
                            git config user.email 'jenkins@tarzan.invalid'

                            git add "$GITOPS_VALUES_FILE"

                            if git diff --cached --quiet; then
                                echo 'GitOps values already reference this image tag.'
                            else
                                git commit -m "deploy(tarzan): image $IMAGE_TAG"
                                git push origin "$GITOPS_BRANCH"
                            fi
                        '''
                    }
                }
            }
        }

        stage('Optional immediate Argo CD sync') {
            when {
                allOf {
                    environment name: 'PUBLISH_RELEASE', value: 'true'
                    expression { params.FORCE_ARGOCD_SYNC }
                }
            }
            steps {
                withCredentials([
                    string(
                        credentialsId: env.ARGOCD_TOKEN_CREDENTIALS_ID,
                        variable: 'ARGOCD_AUTH_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu
                        set +x

                        argocd app sync "$ARGOCD_APP" \
                            --server "$ARGOCD_SERVER" \
                            --auth-token "$ARGOCD_AUTH_TOKEN" \
                            --grpc-web

                        argocd app wait "$ARGOCD_APP" \
                            --server "$ARGOCD_SERVER" \
                            --auth-token "$ARGOCD_AUTH_TOKEN" \
                            --grpc-web \
                            --sync \
                            --health \
                            --timeout 600
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline passed. On the deployment branch, Argo CD owns the rollout of the GitOps commit.'
        }

        failure {
            echo 'Pipeline failed. Check the stage logs for details.'
        }

        always {
            sh '''
                if command -v docker >/dev/null 2>&1; then
                    docker logout "$HARBOR_HOST" \
                        >/dev/null 2>&1 || true
                fi
            '''

            deleteDir()
        }
    }
}
