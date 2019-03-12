podTemplate(label: 'petclinic',
            containers: [
                    containerTemplate(name: 'heptio', image: 'zeppelinops/kubectl-helm-heptio', command: 'cat', ttyEnabled: true),
                    containerTemplate(name: 'docker', image: 'docker', command: 'cat', ttyEnabled: true),
                    containerTemplate(name: 'maven', image: 'maven:3.6.0-jdk-11-slim', command: 'cat', ttyEnabled: true)                         
            ],
            volumes: [
                    hostPathVolume(hostPath: '/maven', mountPath: '/root/.m2'),
                    hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock'),
                    secretVolume(secretName: 'kubeconfig', mountPath: '/home/jenkins/.kube')
            ]) 
{
    node('petclinic') {
        properties([disableConcurrentBuilds()])
        try{
        stage('Checkout') {
            checkout scm
        }
        stage('UnitTest') {
            if(env.BRANCH_NAME == "development") {
                container('maven') {
                    sh 'unset MAVEN_CONFIG && ./mvnw test'
                }
            }
        }
        stage('Build package') {
            container('maven') {
                sh 'unset MAVEN_CONFIG && ./mvnw package -DskipTests'
            }
        }
        stage('Build docker image') {
            if(env.BRANCH_NAME == "development" || env.BRANCH_NAME == "test" || env.BRANCH_NAME == "master" ) {
                def GIT_COMMIT = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()            
                container('docker') {

                    def SERVICE_NAME = "petclinic-development"
                    def DOCKER_IMAGE_REPO = "zeppelinops/petclinic-development"

                    if (env.BRANCH_NAME == 'test') {
                        SERVICE_NAME = "petclinic-test"
                        DOCKER_IMAGE_REPO = "zeppelinops/petclinic-test"
                    }

                    if (env.BRANCH_NAME == 'master') {
                        SERVICE_NAME = "petclinic-production"
                        DOCKER_IMAGE_REPO = "zeppelinops/petclinic-production"
                    }

                    sh """
                        docker build . -t ${SERVICE_NAME}:${GIT_COMMIT} --network host
                        docker tag ${SERVICE_NAME}:${GIT_COMMIT} ${DOCKER_IMAGE_REPO}:${GIT_COMMIT}
                        docker tag ${SERVICE_NAME}:${GIT_COMMIT} ${DOCKER_IMAGE_REPO}:latest
                        """
                    withDockerRegistry(credentialsId: 'docker-hub', url: 'https://index.docker.io/v1/') { 
                        sh "docker push ${DOCKER_IMAGE_REPO}:${GIT_COMMIT}"
                        sh "docker push ${DOCKER_IMAGE_REPO}:latest"
                    }
                }
            }             
        }
        stage('Deploy') { 
            def envMap = [
                development: 'dev',
                test: 'test',                    
                master: 'prod'
            ]
            def env_x = envMap[env.BRANCH_NAME] 
            def GIT_COMMIT = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
            def GIT_COMMIT_MESSAGE = sh(returnStdout: true, script: 'git log -1 --pretty=%B').trim()
            container('heptio') {
                    if(env.BRANCH_NAME == "development" || env.BRANCH_NAME == "test" || env.BRANCH_NAME == "master" ) {
                    dir('.helm') {
                        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-cred']]) {
                            sh """
                                export KUBECONFIG=/home/jenkins/.kube/kubeconfig                            
                                set +e
                                helm upgrade petclinic-${env_x} . -f values-${env_x}.yaml --set image.tag=${GIT_COMMIT} --install --wait --force
                                export DEPLOY_RESULT=\$?
                                [ \$DEPLOY_RESULT -eq 1 ] && helm rollback petclinic-${env_x} 0 && exit 1
                                set -e
                            """
                        }
                    }
                }
            } 
        }
        stage("FunctionalTest"){
            if(env.BRANCH_NAME == "test" || env.BRANCH_NAME == "master" ) {
                echo("Testinium functional tests");
                testiniumExecution failOnTimeout: true, planId: 2979, projectId: 1659, timeoutSeconds: 600
            }
        }
        }
        finally {
        }
    }
}
