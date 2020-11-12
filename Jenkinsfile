pipeline{
    agent any
    environment{
        MYSQL_DATABASE_HOST = "database-42.cbanmzptkrzf.us-east-1.rds.amazonaws.com"
        MYSQL_DATABASE_PASSWORD = "Clarusway"
        MYSQL_DATABASE_USER = "admin"
        MYSQL_DATABASE_DB = "phonebook"
        MYSQL_DATABASE_PORT = 3306
        PATH="/usr/local/bin/:${env.PATH}"
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
                sh "docker build -t ahmet/jenkins ."
                sh "docker tag ahmet/jenkins:latest 931960010427.dkr.ecr.us-east-1.amazonaws.com/ahmet/jenkins:latest"
            }
        }
        stage('push'){
            agent any
            steps{
                sh "aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 931960010427.dkr.ecr.us-east-1.amazonaws.com"
                sh "docker push 931960010427.dkr.ecr.us-east-1.amazonaws.com/ahmet/jenkins:latest"
            }
        }
        stage('compose'){
            agent any
            steps{
                sh "aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 046402772087.dkr.ecr.us-east-1.amazonaws.com"
                sh "docker-compose up -d"
            }
        }

        stage('get-keypair'){
            agent any
            steps{
                sh '''
                    if [ -f "MyKeyPairAK.pem" ]
                    then
                        echo "file exists..."
                    else
                        aws ec2 create-key-pair \
                          --region us-east-1 \
                          --key-name MyKeyPairAK.pem \
                          --query KeyMaterial \
                          --output text > MyKeyPairAK.pem
                        chmod 400 MyKeyPairAK.pem
                        ssh-keygen -y -f MyKeyPairAK.pem >> MyKeyPairAK.pem_public.pem
                    fi
                '''
            }
        }

                stage('create-cluster'){
            agent any
            steps{
                // dont forget to add jenkins to sudoers ----> jenkins ALL=(ALL) NOPASSWD: ALL
                sh '''
                    #!/bin/sh
                    echo $HOME
                    running=$(sudo lsof -i:80) || true
                    if [ "$running" != '' ]
                    then
                        docker-compose down
                        exist="$(aws eks list-clusters | grep ahmet-cluster)" || true
                        if [ "$exist" == '' ]
                        then
                            eksctl create cluster \
                                --name ahmet-cluster \
                                --version 1.18 \
                                --region us-east-1 \
                                --nodegroup-name ahmet-nodes \
                                --node-type t2.small \
                                --nodes 1 \
                                --nodes-min 1 \
                                --nodes-max 2 \
                                --ssh-access \
                                --ssh-public-key  MyKeyPairAK.pem_public.pem \
                                --managed
                        else
                            echo 'no need to create cluster...'
                        fi
                    else
                        echo 'app is not running with docker-compose up -d'
                    fi
                '''
            }
        }
        
    }
}
