pipeline {
    agent any

    environment {
        // 1. 환경 변수 설정
        AWS_REGION = 'ap-northeast-2'
        ECR_REGISTRY = '639242342449.dkr.ecr.ap-northeast-2.amazonaws.com'
        ECR_REPOSITORY = 'jenkins'
        
        // 태그: 젠킨스 빌드 번호 사용 (v1, v2...)
        IMAGE_TAG = "v${BUILD_NUMBER}" 
        
        // Git 설정
        GIT_URL = 'https://github.com/JEONGHO0316/Final_project_Jenkins.git'
        GIT_BRANCH = 'main'
        GITHUB_CRED_ID = 'github-access-token' 
    }

    stages {
        stage('Checkout') {
            steps {
                // 코드 가져오기
                git branch: "${GIT_BRANCH}",
                    credentialsId: "${GITHUB_CRED_ID}",
                    url: "${GIT_URL}"
            }
        }

        stage('Build & Push to ECR') {
            steps {
                script {
                    // AWS ECR 로그인 & Docker 작업
                    withCredentials([[
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-ecr-credentials',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    ]]) {
                        // ECR 로그인
                        sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"
                        
                        // Maven 빌드 (권한 부여 후 실행)
                        sh "chmod +x mvnw"
                        sh "./mvnw clean package -DskipTests"

                        // Docker 빌드 & 푸시
                        sh "docker build -t ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG} ."
                        sh "docker push ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}"
                        
                        // latest 태그도 업데이트 (선택사항)
                        sh "docker tag ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG} ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest"
                        sh "docker push ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest"
                    }
                }
            }
        }

        stage('Update Manifest & Git Push') {
            steps {
                script {
                    // Git 설정 (봇처럼 보이게)
                    sh "git config user.email 'jenkins@example.com'"
                    sh "git config user.name 'Jenkins CI'"

                    // YAML 파일 수정 (sed 사용)
                    // 주의: 기존 이미지 주소를 찾아서 새 태그로 변경
                    sh "sed -i 's|image: .*${ECR_REPOSITORY}:.*|image: ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}|g' k8s/was-deployment.yaml"
                    
                    // 수정 확인
                    sh "cat k8s/was-deployment.yaml"

                    // Git Commit & Push
                    withCredentials([usernamePassword(credentialsId: "${GITHUB_CRED_ID}", passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                        sh "git add k8s/was-deployment.yaml"
                        sh "git commit -m 'GitOps: Deploy new image version ${IMAGE_TAG} by Jenkins'"
                        // 인증 정보 포함해서 Push
                        sh "git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/JEONGHO0316/Final_project_Jenkins.git HEAD:${GIT_BRANCH}"
                    }
                }
            }
        }
    }
}
