pipeline {
    agent {
        // label 'my-slave'
        label 'k8s-slave'
    }
    parameters {
        choice(name: 'scanOnly', choices: 'no\nyes', description: 'Run Sonarqube scans only')
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

        // docker.io/devopswithcloudhub/i27-helpdesk-ui:tagname
        // docker.io/devopswithcloudhub/i27-helpdesk-ui:84285da

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
                expression {
                    return params.BUILD
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
                    expression { params.skipScans == 'no'}
                    anyOf {
                        expression { params.scanOnly == 'yes'}
                        expression { params.buildOnly == 'yes'}
                        expression { params.dockerPush == 'yes'}
                    }
                }
                expression { params.skipScans == 'no'}
                anyOf {
                    expression { params.scanOnly == 'yes'}
                    expression { params.buildOnly == 'yes'}
                    expression { params.dockerPush == 'yes'}
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
    }
}