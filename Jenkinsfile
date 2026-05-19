def deployToGKE(String namespace, String envLabel) {
    env.NAMESPACE = namespace
    sh """
        echo "******************* Deploying to ${envLabel} Environment *********************"
        echo "Deploying into this namespace: ${NAMESPACE}"
        kubectl get pods -n ${NAMESPACE}
        # Substitute variables in kubernetes manifests
        sed -i "s|\\\${NAMESPACE}|${NAMESPACE}|g" k8s/*.yaml
        sed -i "s|\\\${IMAGE_NAME}|${IMAGE_NAME}|g" k8s/deploy.yaml
        sed -i "s|\\\${IMAGE_TAG}|${GIT_COMMIT}|g" k8s/deploy.yaml
        echo "Applying k8s manifests in ${envLabel} namespace"
        kubectl apply -f k8s/
        echo "Deployment to ${envLabel} namespace is completed"
    """

}
def gkeAuth(String clusterName, String zone, String projectId){
    sh """
        echo "******************************* Authenticating to GKE **************************"
        gcloud container clusters get-credentials ${clusterName} --zone ${zone} --project ${projectId}
        echo "******************** Validating the Cluster access *********************"
        kubectl get nodes
    """
}
pipeline {
    agent {
        // label 'my-slave'
        label 'k8s-slave'
    }
    parameters {
        choice(name: 'buildOnly', choices: 'no\nyes', description: 'Build Docker image only, no push')
        choice(name: 'dockerPush', choices: 'no\nyes', description: 'Build and Push Docker image')
        choice(name: 'deployToDev', choices: 'no\nyes', description: 'Deploy to Dev Environment')
        choice(name: 'deployToTest', choices: 'no\nyes', description: 'Deploy to Test Environment')
        choice(name: 'deployToStage', choices: 'no\nyes', description: 'Deploy to Stage Environment')
        choice(name: 'deployToProd', choices: 'no\nyes', description: 'Deploy to Prod Environment')
        choice(name: 'skipScans', choices: 'no\nyes', description: 'Skip Sonarscan and quality gate')
    }
    environment {
        // Currently i am using docker hub registry
        REGISTRY_URL = "docker.io"
        // later u changed to jfrog ====> jfrog.hsbc.com
        IMAGE_REPOSITORY = "devopswithcloudhub/i27-helpdesk-gateway"
        // calling my docker creds into a variable
        REGISTRY_CREDENTIALS_ID = credentials('docker-credentials')

        // Kubernetes Dev Cluster Details 
        DEV_CLUSTER_NAME = "np-cluster"
        DEV_CLUSTER_ZONE = "us-east4-a"
        DEV_PROJECT_ID = "project-fe6816d0-c7fc-4c9b-bd7"

        // Kubernetes Test Cluster Details 
        TEST_CLUSTER_NAME = "np-cluster"
        TEST_CLUSTER_ZONE = "us-east4-a"
        TEST_PROJECT_ID = "project-fe6816d0-c7fc-4c9b-bd7"

        // Kubernetes Stage Cluster Details 
        STAGE_CLUSTER_NAME = "np-cluster"
        STAGE_CLUSTER_ZONE = "us-east4-a"
        STAGE_PROJECT_ID = "project-fe6816d0-c7fc-4c9b-bd7"

        // Kubernetes Prod Cluster Details 
        PROD_CLUSTER_NAME = "np-cluster"
        PROD_CLUSTER_ZONE = "us-east4-a"
        PROD_PROJECT_ID = "project-fe6816d0-c7fc-4c9b-bd7"

 
    }
    stages {
        stage ('Prepare Tag') {
            when {
                anyOf {
                    expression { params.buildOnly == 'yes'}
                    expression { params.dockerPush == 'yes'}
                    expression { params.deployToDev == 'yes'}
                    expression { params.deployToTest == 'yes'}
                    expression { params.deployToStage == 'yes'}
                    expression { params.deployToProd == 'yes'}
                }
            }
            steps {
                script {
                // Construct a full image with complete image name 
                env.IMAGE_NAME = env.REGISTRY_URL + "/" + env.IMAGE_REPOSITORY 

                echo "Using Registry: ${env.REGISTRY_URL}"
                echo "Using Repository: ${env.IMAGE_REPOSITORY}"
                echo "Using Image Tag: ${GIT_COMMIT}"
                echo "Full Image is: ${env.IMAGE_NAME}:${GIT_COMMIT}"
                //docker.io/devopswithcloudhub/i27-helpdesk-ui:COMMIT_ID
                }
            }
        }
        stage ('Sonarqube') {
            when {
                allOf {
                    expression { params.skipScans == 'no' }
                    anyOf {
                        expression { params.buildOnly == 'yes' }
                        expression { params.dockerPush == 'yes' }
                    }
                }
            }
            steps {
                script {
                   def scannerHome = tool 'SonarQubeScanner'
                   withSonarQubeEnv('SonarQube'){
                    sh "${scannerHome}/bin/sonar-scanner"
                   }
                }
                timeout(time: 2, unit: 'MINUTES') {
                    script {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }
        stage('Build Docker Image') {
            when {
                expression { params.buildOnly == 'yes' }
                expression { params.dockerPush == 'yes'}
            }
            steps {
                echo "Building the image"
                sh "docker build --no-cache -t  ${env.IMAGE_NAME}:${GIT_COMMIT} ."
            }
        }
        stage ('Push Image') {
            when {
                expression { params.dockerPush == 'yes'}
            }
            steps {
                echo "********************************* Docker Login *****************************"
                sh "docker login -u ${env.REGISTRY_CREDENTIALS_ID_USR} -p ${env.REGISTRY_CREDENTIALS_ID_PSW} ${env.REGISTRY_URL}"
                echo "********************************* Docker Push *****************************"
                sh "docker push ${env.IMAGE_NAME}:${GIT_COMMIT}"
            }
        }
        stage ('DeployToDevEnvironment'){
            when {
                expression {
                    params.deployToDev == 'yes'
                }
            }
            steps {
                script {
                    // Calling Auth method
                    // String clusterName, String zone, String projectId
                    gkeAuth(env.DEV_CLUSTER_NAME, env.DEV_CLUSTER_ZONE, env.DEV_PROJECT_ID)
                    // Calling deployToEnv method
                    deployToGKE('i27-helpdesk-dev', 'Dev')
                }
            }
        }
    }
}