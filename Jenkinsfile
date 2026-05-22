pipeline {

    agent any

    environment {

        AWS_ACCOUNT_ID = "610405653088"
        AWS_REGION = "ap-south-1"

        FRONTEND_REPO = "streamingapp-frontend"
        AUTH_REPO = "streamingapp-auth"

        ECR_FRONTEND = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${FRONTEND_REPO}"
        ECR_AUTH = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${AUTH_REPO}"

        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Clone Repository') {

            steps {

                git branch: 'dev',
                url: 'https://github.com/hariprn/StreamingApp.git'
            }
        }

        stage('Build Frontend Image') {

            steps {

                sh '''
                docker build \
                --build-arg REACT_APP_AUTH_API_URL=http://a7fd7a95d3f974f9bb66cc5378a6f46b-1763054188.ap-south-1.elb.amazonaws.com:3001/api \
                -t $ECR_FRONTEND:$IMAGE_TAG \
                ./frontend
                '''
            }
        }

        stage('Build Auth Image') {

            steps {

                sh '''
                docker build \
                -t $ECR_AUTH:$IMAGE_TAG \
                ./backend/authService
                '''
            }
        }

        stage('Login to Amazon ECR') {

            steps {

                sh '''
                aws ecr get-login-password --region $AWS_REGION | \
                docker login --username AWS --password-stdin \
                $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
                '''
            }
        }

        stage('Push Frontend Image') {

            steps {

                sh '''
                docker push $ECR_FRONTEND:$IMAGE_TAG
                '''
            }
        }

        stage('Push Auth Image') {

            steps {

                sh '''
                docker push $ECR_AUTH:$IMAGE_TAG
                '''
            }
        }

        stage('Update Helm Values') {

            steps {

                sh '''
                sed -i "s|image: .*streamingapp-frontend.*|image: $ECR_FRONTEND:$IMAGE_TAG|g" helm/streamingapp-chart/values.yaml

                sed -i "s|image: .*streamingapp-auth.*|image: $ECR_AUTH:$IMAGE_TAG|g" helm/streamingapp-chart/values.yaml
                '''
            }
        }

        stage('Deploy to EKS using Helm') {

            steps {

                sh '''
                helm upgrade --install streamingapp \
                helm/streamingapp-chart \
                -n streamingapp
                '''
            }
        }
    }

    post {

    	success {
	
       		 sh '''
       		 aws sns publish \
       		 --topic-arn arn:aws:sns:ap-south-1:610405653088:streamingapp-alerts \
       		 --subject "Jenkins Pipeline Success" \
       		 --message "StreamingApp deployment completed successfully."
       		 '''

       		 echo 'Deployment Successful!'
    	}

    	failure {

       		 sh '''
       		 aws sns publish \
       		 --topic-arn arn:aws:sns:ap-south-1:610405653088:streamingapp-alerts \
       		 --subject "Jenkins Pipeline Failed" \
       		 --message "StreamingApp deployment failed. Check Jenkins logs."
       		 '''

       		 echo 'Pipeline Failed!'
    	}
    }
}
