pipeline {
    agent any

    environment {
        VENV_DIR = 'venv'
        GCP_PROJECT = 'anime-recommender-mlops'
        GCLOUD_PATH = "/var/jenkins_home/google-cloud-sdk/bin"
        KUBECTL_AUTH_PLUGIN = "/usr/lib/google-cloud-sdk/bin"

        COMET_API_KEY = credentials('comet-api-key')
        COMET_PROJECT_NAME = credentials('comet-project-name') 
        COMET_WORKSPACE = credentials('comet-workspace')
    }

    stages{

        stage("Cloning from Github...."){
            steps{
                script{
                    echo 'Cloning from Github...'
                    checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'github-token', url: 'https://github.com/WeOnlyLiveOnce13/MLOps---anime-recommender.git']])
                }
            }
        }

        stage("Making a virtual environment...."){
            steps{
                script{
                    echo 'Building the virtual environment...'
                    sh '''
                    python -m venv ${VENV_DIR}
                    . ${VENV_DIR}/bin/activate
                    pip install --upgrade pip
                    pip install -e .
                    pip install  dvc
                    '''
                }
            }
        }

        stage("Pulling artifacts saved by DVC from GCP bucket into venv"){
            steps{
                withCredentials([file(credentialsId:'gcp-bucket-key' , variable: 'GOOGLE_APPLICATION_CREDENTIALS' )]){
                    script{
                        echo 'DVC Pul....'
                        sh '''
                        . ${VENV_DIR}/bin/activate
                        dvc pull
                        '''
                    }
                }
            }
        }

        stage('Build and Push Image to GCR'){
            steps{
                withCredentials([
                  file(credentialsId:'gcp-bucket-key' , variable: 'GOOGLE_APPLICATION_CREDENTIALS'),
                  string(credentialsId: 'comet-api-key', variable: 'COMET_API_KEY'),
                  string(credentialsId: 'comet-project-name', variable: 'COMET_PROJECT_NAME'),
                  string(credentialsId: 'comet-workspace', variable: 'COMET_WORKSPACE')
                  
                ]){
                    script{
                        echo 'Build and Push Image to GCR'
                        sh '''
                        export PATH=$PATH:${GCLOUD_PATH}
                        gcloud auth activate-service-account --key-file=${GOOGLE_APPLICATION_CREDENTIALS}
                        gcloud config set project ${GCP_PROJECT}
                        gcloud auth configure-docker --quiet
                        docker build \
                            --build-arg COMET_API_KEY=${COMET_API_KEY} \
                            --build-arg COMET_PROJECT_NAME=${COMET_PROJECT_NAME} \
                            --build-arg COMET_WORKSPACE=${COMET_WORKSPACE} \
                            -t gcr.io/${GCP_PROJECT}/ml-project:latest .

                        docker push gcr.io/${GCP_PROJECT}/ml-project:latest
                        '''
                    }
                }
            }
        }

        stage('Deploying to Kubernetes'){
            steps{
                withCredentials([
                    
                    file(credentialsId:'gcp-bucket-key' , variable: 'GOOGLE_APPLICATION_CREDENTIALS'),
                    string(credentialsId: 'comet-api-key', variable: 'COMET_API_KEY'),
                    string(credentialsId: 'comet-project-name', variable: 'COMET_PROJECT_NAME'),
                    string(credentialsId: 'comet-workspace', variable: 'COMET_WORKSPACE')
                    
                ]){
                    script{
                        echo 'Deploying to Kubernetes'
                        sh '''
                        export PATH=$PATH:${GCLOUD_PATH}:${KUBECTL_AUTH_PLUGIN}
                        gcloud auth activate-service-account --key-file=${GOOGLE_APPLICATION_CREDENTIALS}
                        gcloud config set project ${GCP_PROJECT}
                        gcloud container clusters get-credentials ml-app-cluster --region africa-south1

                        kubectl set env deployment/ml-app-deployment \
                            COMET_API_KEY=${COMET_API_KEY} \
                            COMET_PROJECT_NAME=${COMET_PROJECT_NAME} \
                            COMET_WORKSPACE=${COMET_WORKSPACE}
                            
                        envsubst < deployment.yaml | kubectl apply -f -   
                        kubectl apply -f deployment.yaml
                        '''
                    }
                }
            }
        }

    }
}