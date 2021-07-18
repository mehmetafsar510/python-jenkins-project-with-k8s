pipeline{
    agent any
    environment{
        MYSQL_DATABASE_HOST = "mysql-instance.ct3fdmwzcn45.us-east-1.rds.amazonaws.com"
        MYSQL_DATABASE_PASSWORD = "Clarusway"
        MYSQL_DATABASE_USER = "admin"
        MYSQL_DATABASE_DB = "phonebook"
        MYSQL_DATABASE_PORT = 3306
        PATH="/usr/local/bin/:${env.PATH}"
        ECR_REGISTRY = "646075469151.dkr.ecr.us-east-1.amazonaws.com"
        APP_REPO_NAME= "phonebook/app"
        CFN_KEYPAIR="the_doctor"
        AWS_REGION = "us-east-1"
        CLUSTER_NAME = "mehmet-cluster"
        GIT_FOLDER = sh(script:'echo ${GIT_URL} | sed "s/.*\\///;s/.git$//"', returnStdout:true).trim()
    }
    stages{
        stage("compile"){
           agent{
               docker{
                   image 'python:alpine'
               }
           }
           steps{
               withEnv(["HOME=${env.WORKSPACE}"]) {
                    sh 'pip install -r requirements.txt'
                    sh 'python -m py_compile src/*.py'
                    stash(name: 'compilation_result', includes: 'src/*.py*')
                }
           }
        }
        

        stage('create phonebook table in rds'){
            agent any
            steps{
                sh """
                mysql -u ${MYSQL_DATABASE_USER} -h ${MYSQL_DATABASE_HOST} -p${MYSQL_DATABASE_PASSWORD} << `EOF'
                USE ${MYSQL_DATABASE_DB};
                CREATE TABLE IF NOT EXISTS phonebook.phonebook(
                id INT NOT NULL AUTO_INCREMENT,
                name VARCHAR(100) NOT NULL,
                number VARCHAR(100) NOT NULL,
                PRIMARY KEY (id)
                ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
                exit;
                `EOF'
                """
            }
        } 
       
        stage('test'){
            agent {
                docker {
                    image 'python:alpine'
                }
            }
            steps {
                withEnv(["HOME=${env.WORKSPACE}"]) {
                    sh 'python -m pytest -v --junit-xml results.xml src/appTest.py'
                }
            }
            post {
                always {
                    junit 'results.xml'
                }
            }
        }  

        stage('creating .env for docker-compose'){
            agent any
            steps{
                script {
                    echo 'creating .env for docker-compose'
                    sh "cd ${WORKSPACE}"
                    writeFile file: '.env', text: "ECR_REGISTRY=${ECR_REGISTRY}\nAPP_REPO_NAME=${APP_REPO_NAME}:latest"
                }
            }
        }

        stage('creating ECR Repository') {
            steps {
                echo 'creating ECR Repository'
                sh """
                aws ecr create-repository \
                  --repository-name ${APP_REPO_NAME} \
                  --image-scanning-configuration scanOnPush=false \
                  --image-tag-mutability MUTABLE \
                  --region ${AWS_REGION}
                """
            }
        } 

        stage('build'){
            agent any
            steps{
                sh "docker build -t ${APP_REPO_NAME} ."
                sh 'docker tag ${APP_REPO_NAME} "$ECR_REGISTRY/$APP_REPO_NAME:latest"'
            }
        }

        stage('push'){
            agent any
            steps{
                sh 'aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin "$ECR_REGISTRY"'
                sh 'docker push "$ECR_REGISTRY/$APP_REPO_NAME:latest"'
            }
        }

        stage('compose'){
            agent any
            steps{
                sh 'aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin "$ECR_REGISTRY"'
                sh "docker-compose up -d"
            }
        }

        stage('get-keypair'){
            agent any
            steps{
                sh '''
                    if [ -f "${CFN_KEYPAIR}.pem" ]
                    then 
                        echo "file exists..."
                    else
                        aws ec2 create-key-pair \
                          --region us-east-1 \
                          --key-name ${CFN_KEYPAIR}.pem \
                          --query KeyMaterial \
                          --output text > ${CFN_KEYPAIR}.pem

                        chmod 400 ${CFN_KEYPAIR}

                        ssh-keygen -y -f ${CFN_KEYPAIR}.pem >> the_doctor_public.pem
                    fi
                '''                
            }
        }

        stage('create-cluster'){
            agent any
            steps{
                sh "eksctl create cluster --region ${AWS_REGION} --node-type t2.medium --nodes 1 --nodes-min 1 --nodes-max 2 --node-volume-size 8 --name ${CLUSTER_NAME}"
            }
        }

        stage('Test the cluster') {
            steps {
                echo "Testing if the K8s cluster is ready or not"
            script {
                while(true) {
                    try {
                      sh "kubectl get nodes | grep -i Ready"
                      echo "Successfully created  EKS cluster."
                      break
                    }
                    catch(Exception) {
                      echo 'Could not get cluster please wait'
                      sleep(5)   
                    }
                }
            }
        }
    }

        stage('check-cluster'){
            agent any
            steps{
                sh '''
                    #!/bin/sh
                    running=$(sudo lsof -i:80) || true
                    
                    if [ "$running" != '' ]
                    then
                        docker-compose down
                        exist="$(eksctl get cluster | grep mehmet-cluster)" || true

                        if [ "$exist" == '' ]
                        then
                            
                            echo "we alreay created this cluster...."
                        else
                            echo 'no need to create cluster...'
                        fi
                    else
                        echo 'app is not running with docker-compose up -d'
                    fi
                '''
            }
        }

        stage('create-ebs'){
            agent any
            steps{
                sh '''
                    VolumeId=$(aws ec2 describe-volumes --filters Name=tag:Name,Values="k8s-python-mysql2" | grep VolumeId |cut -d '"' -f 4| head -n 1)  || true
                    if [ "$VolumeId" == '' ]
                    then
                        aws ec2 create-volume \
                            --availability-zone us-east-1c\
                            --volume-type gp2 \
                            --size 10 \
                            --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=k8s-python-mysql2}]'
                        
                    fi
                '''
            }
        }

        stage('apply-k8s'){
            agent any
            steps{
                script {
                    env.EBS_VOLUME_ID = sh(script:"aws ec2 describe-volumes --filters Name=tag:Name,Values='k8s-python-mysql2' | grep VolumeId |cut -d '\"' -f 4| head -n 1", returnStdout: true).trim()
                }
                sh "sed -i 's/{{EBS_VOLUME_ID}}/$EBS_VOLUME_ID/g' k8s/pv-ebs.yaml"
                sh "sed -i 's/{{ECR_REGISTRY}}/$ECR_REGISTRY/$APP_REPO_NAME:latest/g' k8s/deployment-app.yaml"
                sh "kubectl apply -f k8s"                
            }
        }
    
    }
    post {
        always {
            echo 'Deleting all local images'
            sh 'docker image prune -af'
        }
        failure {
            sh "rm -rf '${WORKSPACE}/.env'"
            sh "eksctl delete cluster ${CLUSTER_NAME}"
            sh """
            aws ecr delete-repository \
              --repository-name ${APP_REPO_NAME} \
              --region ${AWS_REGION}\
              --force
            """
            sh """
            aws rds delete-db-instance \
              --db-instance-identifier mysql-instance \
              --skip-final-snapshot \
              --delete-automated-backups
            """
            sh "kubectl delete -f k8s"
        }
        success {
            echo 'You are Greattt...'
        }
    }
}

