
pipeline {

    agent any

    environment {
        AWS_REGION      = 'us-east-1'
        AWS_ACCOUNT_ID  = '570367131376'
        ECR_REPOSITORY  = 'ecs-devops-repo'

        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

        ECS_CLUSTER     = 'ecs-devops-cluster'
        ECS_SERVICE     = 'ecs-devops-service'
        ECS_TASK_FAMILY = 'ecs-devops-task'

        CONTAINER_NAME = 'ecs-devops'

        IMAGE_TAG  = "${BUILD_NUMBER}"
        IMAGE_NAME = "${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}"
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
                    docker build -t "${IMAGE_NAME}" .
                '''
            }
        }

        stage('ECR Login') {
            steps {
                sh '''
                    aws ecr get-login-password \
                        --region "${AWS_REGION}" |
                    docker login \
                        --username AWS \
                        --password-stdin "${ECR_REGISTRY}"
                '''
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh '''
                    echo "Pushing image: ${IMAGE_NAME}"
                    docker push "${IMAGE_NAME}"
                '''
            }
        }

        stage('Prepare ECS Task Definition') {
            steps {
                sh '''
                    aws ecs describe-task-definition \
                        --task-definition "${ECS_TASK_FAMILY}" \
                        --region "${AWS_REGION}" \
                        --query taskDefinition \
                        --output json > taskdef.json

                    jq --arg IMAGE "${IMAGE_NAME}" \
                       --arg CONTAINER "${CONTAINER_NAME}" '
                        del(
                            .taskDefinitionArn,
                            .revision,
                            .status,
                            .requiresAttributes,
                            .compatibilities,
                            .registeredAt,
                            .registeredBy,
                            .deregisteredAt
                        )
                        |
                        if any(.containerDefinitions[]; .name == $CONTAINER)
                        then
                            .containerDefinitions |= map(
                                if .name == $CONTAINER
                                then .image = $IMAGE
                                else .
                                end
                            )
                        else
                            error("Container name not found in task definition")
                        end
                    ' taskdef.json > taskdef-new.json
                '''
            }
        }

        stage('Register ECS Task Definition') {
            steps {
                sh '''
                    aws ecs register-task-definition \
                        --cli-input-json file://taskdef-new.json \
                        --region "${AWS_REGION}" \
                        --query 'taskDefinition.taskDefinitionArn' \
                        --output text > new-task-definition-arn.txt

                    echo "Registered task definition:"
                    cat new-task-definition-arn.txt
                '''
            }
        }

        stage('Deploy to ECS') {
            steps {
                sh '''
                    NEW_TASK_DEF=$(cat new-task-definition-arn.txt)

                    echo "Updating ECS service to: ${NEW_TASK_DEF}"

                    aws ecs update-service \
                        --cluster "${ECS_CLUSTER}" \
                        --service "${ECS_SERVICE}" \
                        --task-definition "${NEW_TASK_DEF}" \
                        --region "${AWS_REGION}"

                    echo "Waiting for ECS service to become stable..."

                    aws ecs wait services-stable \
                        --cluster "${ECS_CLUSTER}" \
                        --services "${ECS_SERVICE}" \
                        --region "${AWS_REGION}"

                    echo "ECS deployment completed successfully!"
                '''
            }
        }

        stage('Deployment Verification') {
            steps {
                sh '''
                    aws ecs describe-services \
                        --cluster "${ECS_CLUSTER}" \
                        --services "${ECS_SERVICE}" \
                        --region "${AWS_REGION}" \
                        --query 'services[0].{
                            Status:status,
                            Desired:desiredCount,
                            Running:runningCount,
                            Pending:pendingCount,
                            TaskDefinition:taskDefinition,
                            Deployments:deployments[*].rolloutState
                        }' \
                        --output json
                '''
            }
        }
    }

    post {
        success {
            echo 'DEPLOYMENT SUCCESSFUL'
            echo "Docker Image: ${env.IMAGE_NAME}"
            echo "ECS Cluster: ${env.ECS_CLUSTER}"
            echo "ECS Service: ${env.ECS_SERVICE}"
        }

        failure {
            echo 'DEPLOYMENT FAILED'
            echo 'Check Jenkins console output for the failed stage.'
        }

        always {
            sh '''
                rm -f taskdef.json
                rm -f taskdef-new.json
                rm -f new-task-definition-arn.txt
            '''
        }
    }
}
