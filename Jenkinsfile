pipeline {

    agent any

    environment {

        AWS_REGION = 'us-east-1'

        AWS_ACCOUNT_ID = '570367131376'

        ECR_REPOSITORY = 'ecs-devops-repo'

        ECR_REGISTRY =
            "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

        // Change these three values to match your Terraform resources
        ECS_CLUSTER = 'ecs-devops-cluster'

        ECS_SERVICE = 'ecs-devops-service'

        ECS_TASK_FAMILY = 'ecs-devops-task'

        // Must match the container "name" in your ECS task definition
        CONTAINER_NAME = 'ecs-devops'
        IMAGE_TAG = '${env.ECR_REGISTRY}/${env.ECR_REPOSITORY}:${BUILD_NUMBER}'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Maven Test') {
            steps {
                sh '''
                    cd app
                    mvn clean test
                '''
            }
        }

        stage('Maven Package') {
            steps {
                sh '''
                    cd app
                    mvn package -DskipTests
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build \
                    -t ${IMAGE_NAME} .
                '''
            }
        }

        stage('ECR Login') {
            steps {
                sh '''
                    aws ecr get-login-password \
                    --region ${AWS_REGION} |
                    docker login \
                    --username AWS \
                    --password-stdin \
                    ${ECR_REGISTRY}
                '''
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh '''
                    echo "Pushing image: ${IMAGE_NAME}"

                    docker push ${IMAGE_NAME}
                '''
            }
        }

        stage('Prepare ECS Task Definition') {
            steps {
                sh '''
                    echo "Getting current ECS task definition..."

                    aws ecs describe-task-definition \
                        --task-definition ${ECS_TASK_FAMILY} \
                        --region ${AWS_REGION} \
                        --query taskDefinition \
                        --output json > taskdef.json

                    echo "Updating container image..."

                    jq --arg IMAGE "${IMAGE_NAME}" \
                       --arg CONTAINER "${CONTAINER_NAME}" \
                    '
                    del(
                        .taskDefinitionArn,
                        .revision,
                        .status,
                        .requiresAttributes,
                        .compatibilities,
                        .registeredAt,
                        .registeredBy
                    )
                    |
                    .containerDefinitions |=
                    map(
                        if .name == $CONTAINER
                        then .image = $IMAGE
                        else .
                        end
                    )
                    ' taskdef.json > taskdef-new.json
                '''
            }
        }

        stage('Register ECS Task Definition') {
            steps {
                sh '''
                    echo "Registering new ECS task definition..."

                    aws ecs register-task-definition \
                        --cli-input-json file://taskdef-new.json \
                        --region ${AWS_REGION} \
                        > new-task-definition.json

                    cat new-task-definition.json
                '''
            }
        }

       stage('Deploy to ECS') {
    steps {
        sh '''
            echo "Getting current ECS task definition..."

            aws ecs describe-task-definition \
                --task-definition ecs-devops-task \
                --region $AWS_REGION \
                --query taskDefinition \
                --output json > task-definition.json

            echo "Creating new task definition with image: $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG"

            jq --arg IMAGE "$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" \
                '.containerDefinitions[0].image = $IMAGE
                | del(
                    .taskDefinitionArn,
                    .revision,
                    .status,
                    .requiresAttributes,
                    .compatibilities,
                    .registeredAt,
                    .registeredBy
                  )' \
                task-definition.json > new-task-definition.json

            echo "Registering new task definition..."

            NEW_TASK_DEF=$(aws ecs register-task-definition \
                --cli-input-json file://new-task-definition.json \
                --region $AWS_REGION \
                --query 'taskDefinition.taskDefinitionArn' \
                --output text)

            echo "New task definition: $NEW_TASK_DEF"

            echo "Updating ECS service..."

            aws ecs update-service \
                --cluster ecs-devops-cluster \
                --service ecs-devops-service \
                --task-definition $NEW_TASK_DEF \
                --region $AWS_REGION

            echo "Waiting for ECS deployment..."

            aws ecs wait services-stable \
                --cluster ecs-devops-cluster \
                --services ecs-devops-service \
                --region $AWS_REGION

            echo "ECS deployment completed successfully!"
        '''
    }
}

        stage('Wait for ECS Deployment') {
            steps {
                sh '''
                    echo "Waiting for ECS deployment..."

                    aws ecs wait services-stable \
                        --cluster ${ECS_CLUSTER} \
                        --services ${ECS_SERVICE} \
                        --region ${AWS_REGION}

                    echo "ECS deployment is stable."
                '''
            }
        }

        stage('Deployment Verification') {
            steps {
                sh '''
                    echo "Checking ECS service..."

                    aws ecs describe-services \
                        --cluster ${ECS_CLUSTER} \
                        --services ${ECS_SERVICE} \
                        --region ${AWS_REGION} \
                        --query 'services[0].deployments'
                '''
            }
        }
    }

    post {

        success {
            echo '''
            ==========================================
               DEPLOYMENT SUCCESSFUL
            ==========================================
            '''

            echo "Docker Image: ${env.IMAGE_NAME}"
            echo "ECS Cluster: ${env.ECS_CLUSTER}"
            echo "ECS Service: ${env.ECS_SERVICE}"
        }

        failure {
            echo '''
            ==========================================
               DEPLOYMENT FAILED
            ==========================================
            '''

            echo 'Check Jenkins console output for the failed stage.'
        }

        always {
            sh '''
                rm -f taskdef.json
                rm -f taskdef-new.json
                rm -f new-task-definition.json
            '''
        }
    }
}
