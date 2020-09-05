pipeline {
    
    options { timestamps() }

    environment { 
        cluster_name = 'Capstone-cluster'
        registry = 'javiercaparo/nodejs-image-demo'
		registryCredential = 'dockerhub'
        ekr_registry = '156823553040.dkr.ecr.us-west-2.amazonaws.com'
		dockerImage = ''
        region = 'us-west-2'
        s3_bucket = 's3://jc-eks-cloudformation'
        CI = 'true'
        app = 'my-app'
    }

    agent any

    tools {nodejs "nodejs" }

    parameters {
        gitParameter name: 'RELEASE_TAG',
        type: 'PT_TAG',
        defaultValue: 'master'
    }

    stages {
        
        stage('Cloning Git') {
            steps {
                step([$class: 'WsCleanup'])
                git 'https://github.com/jfcb853/Udacity-DevOps-Capstone-Project'
            }
        }
        stage('Build Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Dev tests') {
            when {
                not { branch 'master' }
            }
            parallel {
                stage('Eslint Test') {
                    steps {
                        sh 'npm run lint'
                    }
                }
                stage('Prettier Test') {
                    steps {
                        sh 'npm run prettify'
                    }
                }
            }
            
        }

        stage('Code Quality Check via SonarQube') {
            when {
                branch 'master'
            }
            steps {
                script {
                    def scannerHome = tool 'sonarqube-scanner';
                        withSonarQubeEnv("SonarQube") {
                            sh "${tool("sonarqube-scanner")}/bin/sonar-scanner \
                            -Dsonar.projectKey=test-node-js \
                            -Dsonar.sources=. \
                            -Dsonar.css.node=. \
                            -Dsonar.host.url=http://localhost:9000 \
                            -Dsonar.login=ba1fe6e839f20654280304081e68bff83a2bbd57"
                        }
                }
            }
        }

        stage('SonarQube Scan Branch') {
            when {
                not { branch 'master' }
            }
            steps {
                script {
                    def scannerHome = tool 'sonarqube-scanner';
                        withSonarQubeEnv("SonarQube") {
                            sh "${tool("sonarqube-scanner")}/bin/sonar-scanner \
                            -Dsonar.projectKey=branch-test-node-js \
                            -Dsonar.sources=. \
                            -Dsonar.css.node=. \
                            -Dsonar.host.url=http://localhost:9000 \
                            -Dsonar.login=ba1fe6e839f20654280304081e68bff83a2bbd57"
                        }
                }
            }
        }
        
        stage('Lint Dockerfile') {
            steps {
                sh 'hadolint Dockerfile'
            }
        }

        stage('Basic Information') {
            steps {
                sh "echo tag: ${params.RELEASE_TAG}"
            }
        }

        stage('Build Image to dockerhub') {
            when {
                branch 'master'
            }
            steps {
                echo 'build the image tagging'
                withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD']]){
					sh '''
                        docker build -t $registry:$BUILD_NUMBER .
					'''
				}
            }
        }

        stage('Dev branches:Build and deploy to EKR') {
            when {
                not { branch 'master' }
            }
            steps {
				withAWS(credentials: 'aws-credentials', region: 'us-west-2') {
                    echo 'Login first'
                    sh 'aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin $ekr_registry'
                    echo 'Build the image to EKR'
                    sh 'docker build -t eks-webapp:$BRANCH_NAME.$BUILD_NUMBER .'
                    echo 'Tagging the image'
                    sh 'docker tag eks-webapp:$BRANCH_NAME.$BUILD_NUMBER $ekr_registry/eks-webapp:$BRANCH_NAME.$BUILD_NUMBER'
                    echo 'Push the image to EKR repo'
                    sh 'docker push 156823553040.dkr.ecr.us-west-2.amazonaws.com/eks-webapp:$BRANCH_NAME.$BUILD_NUMBER'
                }
			}
        }

		stage('Push Prod Image To Dockerhub') {
			when {
                branch 'master'
            }
            steps {
				withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD']]){
					sh '''
						docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD
						docker push $registry:$BUILD_NUMBER
					'''
				}
			}
		}

        stage('Get the cluster kube config') {
			when {
                branch 'master'
            }
            steps {
				withAWS(credentials: 'aws-credentials', region: 'us-west-2') {
					sh '''
						aws eks --region $region update-kubeconfig --name $cluster_name
						ls ~/.kube/config
                        cat ~/.kube/config
                        kubectl get svc
					'''
				}	
			}
		}

        stage('Deploy to K8s') {
			when {
                branch 'master'
            }
            steps {
				withAWS(credentials: 'aws-credentials', region: 'us-west-2') {
					sh "sed -i 's/latest/${env.BUILD_NUMBER}/g' ./eks/deployment.yaml" 
                    sh '''
                        grep image ./eks/deployment.yaml
                        kubectl apply -f ./eks/deployment.yaml
						sleep 2
                        kubectl get deployments
					'''
				}	
			}
		}

        stage('Service to K8s') {
			when {
                branch 'master'
            }
            steps {
				withAWS(credentials: 'aws-credentials', region: 'us-west-2') {
                    sh '''
                        kubectl get pods -l 'app=my-app' -o wide | awk {'print $1" " $3 " " $6'} | column -t
                        kubectl get deployments
                        kubectl apply -f ./eks/service.yaml
                        kubectl get services

                    '''
				}	
			}
		}

        stage('Info about deployment') {
			when {
                branch 'master'
            }
            steps {
				withAWS(credentials: 'aws-credentials', region: 'us-west-2') {
                    sh '''
                        kubectl get nodes -o wide |  awk {'print $1" " $2 " " $7'} | column -t
                        kubectl get pods -l 'app=my-app' -o wide | awk {'print $1" " $3 " " $6'} | column -t
                        sleep 5
                        kubectl get service/my-app |  awk {'print $1" " $2 " " $4 " " $5'} | column -t
                    '''
				}	
			}
		}

        stage('Wait user check the app ') {
            when {
                branch 'master'
            }
            steps {
                input message: 'Checking your web site is ok? (Click "Proceed" to continue)'
            }
        }

		stage('Remove Image locally') {
			when {
                branch 'master'
            }
            steps{
                sh "docker rmi $registry:$BUILD_NUMBER"
            }
		}

        stage('Remove Image from Devs') {
			when {
                not { branch 'master' }
            }
            steps{
                sh "docker rmi eks-webapp:$BRANCH_NAME.$BUILD_NUMBER"
            }
		}
    }

    post {
        always {
            cleanWs()
            echo 'Pipeline finished'
        }
        cleanup{
            cleanWs()
            echo 'Wipeout workspace'
        }
    }
}