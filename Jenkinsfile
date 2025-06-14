pipeline 
{
    agent any

    parameters {
        string(name: 'AWS_REGION', defaultValue: 'us-east-1', description: 'AWS REGION FOR ECR')
        string(name: 'REPO_DOCKER_USER', defaultValue: 'ribun', description: 'user Docker Hub')
        string(name: 'TAG', defaultValue: 'v1.0', description: 'Tag docker images ex(image:TAG)')
        string(name: 'CLUSTER_NAME', defaultValue: 'Task-Management-Cluster', description: 'Name EKS cluster')
    }
    environment {
        AWS_REGION  = "${params.AWS_REGION}"
        SNYK_TOKEN  = credentials('snyk')

        CLUSTER_NAME  = "${params.CLUSTER_NAME}"
    }
    stages {
        // kiểm tra mã nguồn
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/RibunLoc/jenkins-pipeline-lab.git'
            }
        }
        // Kiểm tra sự thay đổi của services
        stage('Check services') {
            steps {
                script {
                    def changedFiles = sh(script: "git diff --name-only $GIT_PREVIOUS_COMMIT $GIT_COMMIT", returnStdout: true).trim().split("\n")
                    def detectServices = ""
                    
                    if (changedFiles.any { it.startsWith("user-service/auth/")}) {
                        detectServices = "auth"
                    } else if (changedFiles.any { it.startsWith("user-service/user/")}) {
                        detectServices = "user"
                    } else if (changedFiles.any { it.startsWith("task-service/")}) {
                        detectServices = "task"
                    }

                    if ( detectServices == "") {
                        error("Không thể tìm thấy dịch vụ cần update")
                    }
                    env.SERVICES = detectServices
                }
            }
        }
        // Thiết lập biến môi trường chuẩn bị cho quy trình 
        stage('Setup ENV') {
            steps {
                script {
                    def serviceNameMap = [
                        'auth' : 'auth-service',
                        'user' : 'user-service',
                        'task' : 'task-service'
                    ]
                    def serviceMap = [
                            'auth' : 'user-service/auth', 
                            'user' : 'user-service/user',
                            'task' :  'task-service/'
                    ]
                    def portMap = [
                            'auth' : 8080,
                            'user' : 8080,
                            'task' : 8080
                    ]
                    env.SERVICE_NAME = serviceNameMap[env.SERVICES]
                    env.DIR_NAME = serviceMap[env.SERVICES]
                    env.PORT = portMap[env.SERVICES]
                    env.REPO_IMAGE = "${env.SERVICES}-service"
                    env.REPO_NAME = "${params.REPO_DOCKER_USER}/muzique-${env.SERVICES}-service:${params.TAG}"
                }
            }
        }
        // Gửi yêu cầu đánh giá mã nguồn tới Sonarqube
        stage('Analys code - SonarQube') {
            environment {
                SCANNER_HOME = tool 'sonar-scanner'
            }
            steps {
                withSonarQubeEnv('sonar-server') {
                sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Muzique_Backend \
                    -Dsonar.projectKey=Muzique-be
                '''
                }
            }
        }
        // Chờ đợi nhận đánh giá từ sonarqube
        stage('QUALITY GATE') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        // Thực hiện đóng gói service tương ứng thành images
        stage('Docker Build'){
            steps {
                script {
                    withDockerRegistry([credentialsId: 'docker', url: 'https://index.docker.io/v1/']) {
                        dir(env.DIR_NAME) {
                            sh '''
                                docker build -t $REPO_IMAGE .
                                docker tag $REPO_IMAGE $REPO_NAME
                                docker push $REPO_NAME
                            '''
                        }
                    }
                }
            }
        }
        // Scan ???
        stage('Scan Image') {
            parallel {
                stage('Trivy scan image') {
                    steps {
                        sh '''
                            trivy image $REPO_IMAGE > trivy.json
                        '''
                    }
                }
                stage('Snyk scan image') {
                    steps {
                        sh '''
                            snyk container test $REPO_NAME
                            snyk container monitor $REPO_NAME
                        '''
                    }
                }
            }
        }
        // Loại bỏ container để tiết kiệm dung lượng lưu trữ
        stage('Remove container') {
            steps {
                script {
                    dir(env.DIR_NAME)
                    {
                        sh '''
                            docker stop $REPO_IMAGE | true
                            docker rm $REPO_IMAGE | true
                        '''
                    }
                }
            }
        }
        // Triển khai dịch vụ tương ứng lên EKS
        stage('Deploy to k8s') {
            steps {
                withCredentials([
                        [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-key'],
                        [$class: 'FileBinding', credentialsId: 'MYSQL-DOTENV', variable: 'ENV_FILE']
                    ]) {
                    sh """
                        # Tạo confiMap Secret
                        kubectl create configmap ${SERVICES}-env --from-env-file=\$ENV_FILE -o yaml --dry-run=client | kubectl appy -f -
                        # triển khai dịch vụ
                        kubectl apply -f k8s/${SERVICES}-deployment.yaml
                    """
                }
            }
        }
    }
}