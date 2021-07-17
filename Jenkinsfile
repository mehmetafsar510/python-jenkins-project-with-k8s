pipeline{
    agent any
    environment{
        MYSQL_DATABASE_HOST = "database-42.cbanmzptkrzf.us-east-1.rds.amazonaws.com"
        MYSQL_DATABASE_PASSWORD = "Clarusway"
        MYSQL_DATABASE_USER = "admin"
        MYSQL_DATABASE_DB = "phonebook"
        MYSQL_DATABASE_PORT = 3306
        PATH="/usr/local/bin/:${env.PATH}"
        ECR_REGISTRY = "646075469151.dkr.ecr.us-east-1.amazonaws.com"
        APP_REPO_NAME= "phonebook/app"
        CFN_KEYPAIR="the_doctor.pem"
        AWS_REGION = "us-east-1"
        CLUSTER_NAME = "mehmet-cluster"
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

        stage('build'){
            agent any
            steps{
                sh "docker build -t phonebook/app ."
                sh 'docker tag phonebook/app "$ECR_REGISTRY/$APP_REPO_NAME":latest'
            }
        }

        stage('push'){
            agent any
            steps{
                sh 'aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin "$ECR_REGISTRY"'
                sh 'docker push "$ECR_REGISTRY/$APP_REPO_NAME":latest'
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
                    if [ -f "${CFN_KEYPAIR}" ]
                    then 
                        echo "file exists..."
                    else
                        aws ec2 create-key-pair \
                          --region us-east-2 \
                          --key-name ${CFN_KEYPAIR} \
                          --query KeyMaterial \
                          --output text > ${CFN_KEYPAIR}

                        chmod 400 ${CFN_KEYPAIR}

                        ssh-keygen -y -f ${CFN_KEYPAIR} >> the_doctor_public.pem
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
                sh "kubectl apply -f k8s"                
            }
            post {
                failure {
                    sh "kubectl delete -f k8s"
                    sh "eksctl delete cluster ${CLUSTER_NAME}"
                }
            }
        }
    }

}

