pipeline {
    agent any

    environment {
        
        // האיזור ב-AWS (למשל us-east-1)
        AWS_REGION = 'us-east-1' 
        
        // ה-ID של חשבון ה-AWS (מספרים בלבד)
        AWS_ACCOUNT_ID = '992382545251' 
        
        // שם ה-Repository שיצרת ב-ECR
        ECR_REPO_NAME = 'alon-test' 
        
        // כתובת ה-IP הציבורית של שרת ה-EC2
        EC2_INSTANCE_IP = '44.212.65.130' 
        
        // שם המשתמש בשרת (ב-AWS בדרך כלל זה ubuntu או ec2-user)
        EC2_USER = 'ec2-user' 
        
        // שמות ה-Credentials ששמרת בתוך Jenkins
        JENKINS_AWS_CREDS_ID = 'my-aws-credentials'
        JENKINS_SSH_KEY_ID = 'my-ec2-ssh-key'
        
        // הרכבת כתובת ה-Registry המלאה
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_URI = "${ECR_REGISTRY}/${ECR_REPO_NAME}:latest"
    }

    stages {
        
        stage('Checkout') {
            steps {
                // מושך את הקוד מ-Github (קורה אוטומטית אם מריצים מתוך ה-Repo, אבל טוב שיהיה)
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo 'Building Docker image...'
                    // בניית האימג' מה-Dockerfile בתיקייה הנוכחית
                    sh "docker build -t ${IMAGE_URI} ."
                }
            }
        }

        stage('Login & Push to ECR') {
            steps {
                script {
                    echo 'Logging into AWS ECR...'
                    // שימוש בפרטי הגישה של AWS כדי להתחבר ל-ECR
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: JENKINS_AWS_CREDS_ID]]) {
                        sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"
                        
                        echo 'Pushing image to ECR...'
                        sh "docker push ${IMAGE_URI}"
                    }
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                script {
                    echo 'Deploying to EC2...'
                    
                    // שימוש ב-SSH Agent כדי להתחבר לשרת המרוחק
                    sshagent(credentials: [JENKINS_SSH_KEY_ID]) {
                        // 1. התחברות ל-ECR מתוך השרת המרוחק (כדי שיוכל למשוך את האימג')
                        // הערה: דורש ש-AWS CLI יהיה מותקן על ה-EC2
                        sh "ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_INSTANCE_IP} 'aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}'"
                        
                        // 2. משיכת האימג' המעודכן
                        sh "ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_INSTANCE_IP} 'docker pull ${IMAGE_URI}'"
                        
                        // 3. עצירה ומחיקה של קונטיינר ישן (אם קיים) כדי לפנות את הפורט
                        // ה- || true נועד למנוע כישלון אם הקונטיינר לא קיים עדיין
                        sh "ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_INSTANCE_IP} 'docker stop flask-app || true'"
                        sh "ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_INSTANCE_IP} 'docker rm flask-app || true'"
                        
                        // 4. הרצת הקונטיינר החדש (ממפה את פורט 5000 לפורט 80 או 5000 בשרת)
                        sh "ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_INSTANCE_IP} 'docker run -d --name flask-app -p 5000:5000 ${IMAGE_URI}'"
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo 'Deployment finished successfully!'
        }
        failure {
            echo 'Deployment failed.'
        }
    }
}
