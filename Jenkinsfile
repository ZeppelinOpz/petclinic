
podTemplate(label: 'petclinic',
            containers: [
                    containerTemplate(name: 'heptio', image: 'zeppelinops/kubectl-helm-heptio', command: 'cat', ttyEnabled: true),
                    containerTemplate(name: 'docker', image: 'docker', command: 'cat', ttyEnabled: true),
                    containerTemplate(name: 'maven', image: 'maven:3.6.0-jdk-8', command: 'cat', ttyEnabled: true)                         
            ],
            volumes: [
                    hostPathVolume(hostPath: '/maven', mountPath: '/home/jenkins/.m2'),
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
        stage('Build package') {
            container('maven') {
                sh 'unset MAVEN_CONFIG && ./mvnw package'
            }
        }
        stage('Build docker image') {
            def GIT_COMMIT = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
            container('docker') {
                def SERVICE_NAME = "petclinic"
                def DOCKER_IMAGE_REPO = "zeppelinops/petclinic"
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
        stage('Deploy') {  
            def GIT_COMMIT = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
            def GIT_COMMIT_MESSAGE = sh(returnStdout: true, script: 'git log -1 --pretty=%B').trim()
            container('heptio') {
                dir('.helm') {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-cred']]) {
                        sh """
                            export KUBECONFIG=/home/jenkins/.kube/kubeconfig                            
                            set +e
                            helm upgrade petclinic-dev . -f values.yaml --set image.tag=${GIT_COMMIT} --install --wait --force
                            export DEPLOY_RESULT=\$?
                            [ \$DEPLOY_RESULT -eq 1 ] && helm rollback petclinic-dev 0 && exit 1
                            set -e
                        """
                    }
                }
            }
        }
        }
        finally {
        }
    }
}