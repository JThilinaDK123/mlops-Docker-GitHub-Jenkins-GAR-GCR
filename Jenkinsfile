pipeline {
    agent any

    environment {
        VENV_DIR   = 'venv'
        GCP_PROJECT = "mlops-thilina"
        GCLOUD_PATH = "/var/jenkins_home/google-cloud-sdk/bin"
        IMAGE_NAME  = "us-central1-docker.pkg.dev/mlops-thilina/mlops-project/mlops-app-project-01:latest"
    }

    stages {
        stage('Cloning Github repo to Jenkins') {
            steps {
                script {
                    echo 'Cloning Github repo to Jenkins............'
                    checkout scmGit(
                        branches: [[name: '*/main']], 
                        extensions: [], 
                        userRemoteConfigs: [[
                            credentialsId: 'jenkins-github-project-01', 
                            url: 'https://github.com/JThilinaDK123/mlops-project-01.git'
                        ]]
                    )
                }
            }
        }

        stage('Setting up Virtual Environment and Installing Dependencies') {
            steps {
                script {
                    echo 'Setting up Virtual Environment and Installing Dependencies............'
                    sh '''
                    python -m venv ${VENV_DIR}
                    . ${VENV_DIR}/bin/activate
                    pip install --upgrade pip
                    pip install -e .
                    '''
                }
            }
        }

        stage('Building and Pushing Docker Image to GCR') {
            steps {
                withCredentials([file(credentialsId: 'gcp-Jenkins-auth', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                    script {
                        echo 'Building and Pushing Docker Image to GCR.............'
                        sh '''
                        export PATH=$PATH:${GCLOUD_PATH}

                        gcloud auth activate-service-account --key-file="${GOOGLE_APPLICATION_CREDENTIALS}"
                        gcloud config set project ${GCP_PROJECT}
                        gcloud auth configure-docker --quiet

                        docker buildx create --use || true
                        docker buildx inspect --bootstrap

                        # Build and push for linux/amd64
                        docker buildx build --platform linux/amd64 -t ${IMAGE_NAME} . --push

                        '''
                    }
                }
            }
        }

        stage('Train Model') {
            steps {
                withCredentials([file(credentialsId: 'gcp-Jenkins-auth', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                    script {
                        echo 'Running Training Pipeline............'
                        sh '''
                        export GOOGLE_APPLICATION_CREDENTIALS=${GOOGLE_APPLICATION_CREDENTIALS}
                        . ${VENV_DIR}/bin/activate
                        python pipeline/training_pipeline.py
                        '''
                    }
                }
            }
        }

        stage('Deploy to Google Cloud Run'){
            steps{
                withCredentials([file(credentialsId: 'gcp-Jenkins-auth' , variable : 'GOOGLE_APPLICATION_CREDENTIALS')]){
                    script{
                        echo 'Deploy to Google Cloud Run.............'
                        sh '''
                        export PATH=$PATH:${GCLOUD_PATH}

                        gcloud auth activate-service-account --key-file="${GOOGLE_APPLICATION_CREDENTIALS}"

                        gcloud config set project ${GCP_PROJECT}

                        gcloud run deploy ml-project \
                            --image=us-central1-docker.pkg.dev/mlops-thilina/mlops-project/mlops-app-project-01:latest \
                            --platform=managed \
                            --region=us-central1 \
                            --allow-unauthenticated 
                      
                        '''
                    }
                }
            }
        }
    }
}
