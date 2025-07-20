pipeline {
    agent any

    environment {
        APP_NAME = "phonebook"
        APP_REPO_NAME = "jenkins-007-repo/${APP_NAME}-app"
        AWS_REGION = "eu-west-1"
        CLUSTER_URL = "https://172.31.25.200:6443"
    }

    stages {
        stage('Create ECR Repositories') {
            steps {
                withCredentials([[ 
                    $class: 'AmazonWebServicesCredentialsBinding', 
                    credentialsId: 'aws-creds' 
                ]]) {
                    script {
                        def AWS_ACCOUNT_ID = sh(script: 'aws sts get-caller-identity --query Account --output text', returnStdout: true).trim()
                        def ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                        def repos = ["web_server", "result_server"]

                        for (repo in repos) {
                            def full_repo = "jenkins-007-repo/${APP_NAME}-${repo}"
                            sh """
                            aws ecr describe-repositories --region ${AWS_REGION} --repository-name ${full_repo} || \
                            aws ecr create-repository \
                                --repository-name ${full_repo} \
                                --image-scanning-configuration scanOnPush=true \
                                --image-tag-mutability MUTABLE \
                                --region ${AWS_REGION}
                            """
                        }
                    }
                }
            }
        }

        stage('Build and Push Docker Images') {
            steps {
                withCredentials([[ 
                    $class: 'AmazonWebServicesCredentialsBinding', 
                    credentialsId: 'aws-creds' 
                ]]) {
                    script {
                        def AWS_ACCOUNT_ID = sh(script: 'aws sts get-caller-identity --query Account --output text', returnStdout: true).trim()
                        def ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                        def components = ["web_server", "result_server"]

                        for (component in components) {
                            def image_tag = "${ECR_REGISTRY}/jenkins-007-repo/${APP_NAME}-${component}:latest"
                            def dockerfile_path = "images/image_for_${component}/Dockerfile"
                            def context_dir = "images/image_for_${component}"

                            sh """
                            docker build -t ${image_tag} -f ${dockerfile_path} ${context_dir}
                            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                            docker push ${image_tag}
                            """
                        }

                        // Export image tags to be used in envsubst
                        env.RESULT_IMAGE = "${ECR_REGISTRY}/jenkins-007-repo/${APP_NAME}-result_server:latest"
                        env.WEB_IMAGE = "${ECR_REGISTRY}/jenkins-007-repo/${APP_NAME}-web_server:latest"
                    }
                }
            }
        }

        stage('Generate Final Kubernetes YAMLs') {
            steps {
                sh """
                mkdir -p k8s/generated
                export RESULT_IMAGE=${RESULT_IMAGE}
                export WEB_IMAGE=${WEB_IMAGE}
                envsubst < k8s/templates/resultserver-deployment.yaml.template > k8s/generated/resultserver-deployment.yaml
                envsubst < k8s/templates/webserver-deployment.yaml.template > k8s/generated/webserver-deployment.yaml
                """
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([string(credentialsId: 'kube-token', variable: 'KUBE_TOKEN')]) {
                    sh '''
                    echo $KUBE_TOKEN > /tmp/token
                    kubectl --server=${CLUSTER_URL} \
                            --token=$(cat /tmp/token) \
                            --insecure-skip-tls-verify=true \
                            --validate=false \
                            apply -f k8s/generated/
                    '''
                }
            }
        }
    }
}
