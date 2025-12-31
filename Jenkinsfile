pipeline {
    agent any

    environment {
        // --- הגדרות משתנים (לפי מה ששלחת) ---
        
        // האיזור ב-AWS
        AWS_REGION = 'us-east-1' 
        
        // ה-ID של חשבון ה-AWS
        AWS_ACCOUNT_ID = '992382545251' 
        
        // שם ה-Repository שיצרת ב-ECR
        ECR_REPO_NAME = 'alon-test' 
        
        // כתובת ה-IP הציבורית של שרת ה-EC2 (Production)
        EC2_INSTANCE_IP = '44.212.65.130' 
        
        // שם המשתמש בשרת (ec2-user מתאים ל-Amazon Linux)
        EC2_USER = 'ec2-user' 
        
        // שמות ה-Credentials ששמרת בתוך Jenkins
        JENKINS_AWS_CREDS_ID = 'my-aws-credentials'
        JENKINS_SSH_KEY_ID = 'my-ec2-ssh-key'
        
        // הרכבת כתובת ה-Registry המלאה (נוצרת אוטומטית)
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_URI = "${ECR_REGISTRY}/${ECR_REPO_NAME}:latest"
    }

    stages {
        
        stage('Checkout') {
            steps {
                // משיכת הקוד מ-Github
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo 'Building Docker image...'
                    // בניית האימג'
                    sh "docker build -t ${IMAGE_URI} ."
                }
            }
        }

        stage('Login & Push to ECR') {
            steps {
                script {
                    echo 'Logging into AWS ECR...'
                    // שימוש בפרטי הגישה של AWS (Access Key / Secret Key) שהגדרנו ב-Jenkins
                    // הפקודה usernamePassword מחלצת את המפתחות למשתני סביבה כדי ש-AWS CLI יוכל להשתמש בהם
                    withCredentials([usernamePassword(credentialsId: JENKINS_AWS_CREDS_ID, usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        
                        // ביצוע Login ל-Docker מול ה-ECR
                        sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"
                        
                        echo 'Pushing image to ECR...'
                        // העלאת האימג' לענן
                        sh "docker push ${IMAGE_URI}"
                    }
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                script {
                    echo 'Deploying to EC2...'
                    
                    // שימוש ב-SSH Agent עם המפתח הפרטי כדי להתחבר לשרת המרוחק
                    sshagent(credentials: [JENKINS_SSH_KEY_ID]) {
                        
                        // הערה חשובה: הפקודות הבאות רצות על שרת ה-EC2 עצמו.
                        // כדי שהפקודה הראשונה (aws ecr...) תעבוד, לשרת ה-EC2 חייב להיות IAM Role
                        // עם הרשאת AmazonEC2ContainerRegistryReadOnly (או ביצוע aws configure בשרת ידנית).

                        // 1. התחברות ל-ECR מתוך השרת המרוחק
                        sh "ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_INSTANCE_IP} 'aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}'"
                        
                        // 2. משיכת האימג' המעודכן
                        sh "ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_INSTANCE_IP} 'docker pull ${IMAGE_URI}'"
                        
                        // 3. עצירה ומחיקה של קונטיינר ישן (כדי לפנות
