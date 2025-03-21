pipeline {
    agent any

    tools {
        maven 'maven3'
        jdk 'jdk17'
    }

    environment {
        DOCKER_IMAGE = "docker.io/guihadyossry/boardgame-app"
        NEXUS_URL = "http://localhost:8081"
        NEXUS_REPO = "maven-releases"
        SONAR_PROJECT_KEY = "Boardgame"
        KUBE_NAMESPACE = "boardgame-app-cicd"
    }

    stages {
        stage('Cleanup Workspace') {
            steps {
                deleteDir()
            }
        }

        stage('Install Kubectl') {
            steps {
                script {
                    sh '''
                    INSTALL_DIR=$HOME/bin
                    KUBECTL_PATH=$INSTALL_DIR/kubectl
                    mkdir -p $INSTALL_DIR
                    
                    if [ -x "$KUBECTL_PATH" ]; then
                        echo "Kubectl is already installed. Skipping download."
                    else
                        echo "Installing Kubectl..."
                        curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                        chmod +x kubectl
                        mv kubectl $INSTALL_DIR/
                    fi
                    
                    export PATH=$INSTALL_DIR:$PATH
                    echo "export PATH=$INSTALL_DIR:\$PATH" >> ~/.bashrc
                    kubectl version --client
                    '''
                }
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/GuihadYosri/Boardgame-CICD-project.git'
            }
        }

        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }

        stage('Run Unit Tests') {
            steps {
                sh "mvn test surefire-report:report"
            }
        }

        stage('Run Integration Tests') {
            steps {
                sh "mvn verify"
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    withCredentials([string(credentialsId: 'sonarqube', variable: 'SONAR_TOKEN')]) {
                        sh '''
                        mvn sonar:sonar \
                        -Dsonar.projectKey=Boardgame \
                        -Dsonar.projectName="BoardGameApp-CICD" \
                        -Dsonar.host.url=http://localhost:9000 \
                        -Dsonar.login=$SONAR_TOKEN
                        '''
                    }
                }
            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                timeout(time: 3, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build JAR') {
            steps {
                sh "mvn package"
            }
        }

        stage('Update Version & Upload to Nexus') {
            steps {
                sh '''
                NEW_VERSION=$(date +%Y.%m.%d-%H%M%S)
                mvn versions:set -DnewVersion=$NEW_VERSION -DgenerateBackupPoms=false
                mvn deploy
                '''
            }
        }

        stage('Upload Test Reports to Nexus') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'nexus-admin', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                        def BUILD_TIMESTAMP = new Date().format("yyyy.MM.dd-HHmmss") 
                        def REPORT_PATH = "$NEXUS_URL/repository/test-reports/com/javaproject/$BUILD_TIMESTAMP/"

                        sh '''
                        for file in target/surefire-reports/*.xml; do
                            curl -u $NEXUS_USER:$NEXUS_PASS --upload-file $file \
                            ''' + REPORT_PATH + '''$(basename $file)
                        done
                        '''
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $DOCKER_IMAGE .'
            }
        }
        
       stage('Trivy Scan') {
    steps {
        script {
            def scanStatus = sh(script: 'trivy image --scanners vuln --timeout 15m --cache-dir /tmp/.cache/trivy --exit-code 1 --severity CRITICAL --no-progress $DOCKER_IMAGE', returnStatus: true)
            if (scanStatus != 0) {
                input message: 'Trivy scan found vulnerabilities. Do you want to proceed?', ok: 'Proceed'
            }
        }
    }
}


         
        stage('Push Docker Image') {
            steps {
                script {
                    IMAGE_TAG = sh(script: "date +%Y%m%d%H%M%S", returnStdout: true).trim()
                    env.IMAGE_TAG = IMAGE_TAG  // Store it as an environment variable
        
                    sh "docker tag $DOCKER_IMAGE $DOCKER_IMAGE:$IMAGE_TAG"
                    withDockerRegistry([credentialsId: 'docker-hub', url: '']) {
                        sh "docker push $DOCKER_IMAGE:$IMAGE_TAG"
                        sh "docker push $DOCKER_IMAGE:latest"
                    }
                }
            }
        }
        // Manual approval before deploying (Continuous Delivery)
        stage('Approval Required Before Deployment') {
            steps {
                input message: 'Proceed with deployment to Minikube?', ok: 'Deploy'
            }
        }
        
        stage('Deploy to Minikube') {
            steps {
                script {
                    sh '''
                    export PATH=$HOME/bin:$PATH  # Ensure kubectl is available
                    export KUBECONFIG=/var/lib/jenkins/.kube/config
                    kubectl config use-context minikube
                    kubectl apply --validate=false -f k8s/namespace.yaml
                    kubectl apply --validate=false -f k8s/boardgame-deployment.yaml
                    kubectl apply --validate=false -f k8s/boardgame-service.yaml
                    kubectl apply --validate=false -f k8s/service-monitor.yaml
                  
                    '''
                }
            }
        }

stage('Deploy Blue-Green') {
    steps {
        script {
            try {
               sh '''
                    export PATH=$HOME/bin:$PATH
                    export KUBECONFIG=/var/lib/jenkins/.kube/config
                    kubectl config use-context minikube

                    # Deploy Blue and Green environments
                    kubectl apply --validate=false -f k8s/namespace.yaml
                    kubectl apply --validate=false -f k8s/boardgame-blue-deployment.yaml
                    kubectl apply --validate=false -f k8s/boardgame-blue-service.yaml
                    kubectl apply --validate=false -f k8s/boardgame-green-deployment.yaml
                    kubectl apply --validate=false -f k8s/boardgame-green-service.yaml

                    # Deploy Nginx Load Balancer
                    kubectl apply --validate=false -f k8s/nginx-configmap.yaml
                    kubectl apply --validate=false -f k8s/nginx-deployment.yaml
                    kubectl apply --validate=false -f k8s/nginx-service.yaml
                

                # Health check: Add a check to confirm if the deployment was successful
                
                kubectl rollout status deployment/boardgame-blue-deployment -n boardgame-app-cicd
                kubectl rollout status deployment/boardgame-green-deployment -n boardgame-app-cicd
                '''
                
            } catch (Exception e) {
                echo "Deployment failed, rolling back..."
                // Rollback to previous stable state
                sh '''
                kubectl rollout undo deployment/boardgame-blue-deployment -n boardgame-app-cicd
                kubectl rollout undo deployment/boardgame-green-deployment -n boardgame-app-cicd
                '''
                throw e
            }
        }
    }
}

      
      stage('Switch Traffic') {
    steps {
        script {
            sh '''
            export PATH=$HOME/bin:$PATH  
            export KUBECONFIG=/var/lib/jenkins/.kube/config  

            # Get current active service (Blue or Green)
            current_version=$(kubectl get configmap nginx-config -n boardgame-app-cicd -o=jsonpath="{.data.nginx\\.conf}" | grep -o 'server boardgame-app-[a-z]*-service' | awk '{print $2}')

            # Determine new version
            new_version="boardgame-app-blue-service"
            if [ "$current_version" == "boardgame-app-blue-service" ]; then
                new_version="boardgame-app-green-service"
            fi

            # Update the config map using kubectl edit approach
            kubectl get configmap nginx-config -n boardgame-app-cicd -o yaml | \
            sed "s/server $current_version;/server $new_version;/" | \
            kubectl apply -f -

            # Restart Nginx Pod to apply changes
            kubectl delete pod -l app=nginx -n boardgame-app-cicd
            '''
        }
    }
}



    }


    

   post {
    success {
        echo 'Build and Deployment Successful!'
    }
    failure {
        echo 'Build Failed. Checking logs...'

        // Send email or notification about failure
        emailext (
            subject: "Build Failed: ${currentBuild.fullDisplayName}",
            body: "The build failed. Please check the logs for more details: ${BUILD_URL}",
            to: "guihadyosry@gmail.com"
        )

       
    }
}

}
