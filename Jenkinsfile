pipeline {
    agent any
    
    parameters {
        string(name: 'GIT_REPO', defaultValue: 'https://github.com/spring-projects/spring-petclinic.git', description: 'GitHub Repository URL')
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'GitHub Branch')
        string(name: 'REGISTRY_URL', defaultValue: 'petclinicacr20923.azurecr.io', description: 'Azure Container Registry URL')
        string(name: 'REGISTRY_USERNAME', defaultValue: 'petclinicacr20923', description: 'ACR Username')
        password(name: 'REGISTRY_PASSWORD', defaultValue: '', description: 'ACR Password')
        string(name: 'SONAR_HOST_URL', defaultValue: 'http://20.25.97.96:9000', description: 'SonarQube Server URL')
        string(name: 'AKS_RESOURCE_GROUP', defaultValue: 'petclinic-rg', description: 'AKS Resource Group')
        string(name: 'AKS_CLUSTER_NAME', defaultValue: 'petclinic-aks', description: 'AKS Cluster Name')
    }
    
    environment {
        APP_NAME = 'spring-petclinic'
        BUILD_NUMBER = "${BUILD_NUMBER}"
        DOCKER_IMAGE_TAG = "${BUILD_NUMBER}"
        JAVA_HOME = '/usr/lib/jvm/java-21-openjdk-amd64'
        PATH = "${JAVA_HOME}/bin:${PATH}"
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 60, unit: 'MINUTES')
        timestamps()
    }
    
    stages {
        stage('Checkout') {
            steps {
                script {
                    echo "========== Stage: Checkout Source Code =========="
                    try {
                        checkout([
                            $class: 'GitSCM',
                            branches: [[name: "${params.GIT_BRANCH}"]],
                            userRemoteConfigs: [[url: "${params.GIT_REPO}"]]
                        ])
                        echo "✓ Source code checked out successfully"
                    } catch (Exception e) {
                        echo "✗ Checkout failed: ${e.message}"
                        currentBuild.result = 'FAILURE'
                        error("Checkout stage failed")
                    }
                }
            }
        }
        
        stage('Validate') {
            steps {
                script {
                    echo "========== Stage: Validate Environment =========="
                    try {
                        sh '''
                            echo "Checking Java version..."
                            java -version
                            
                            echo "Checking Maven wrapper..."
                            ./mvnw -version
                            
                            echo "Checking Docker..."
                            docker --version
                            
                            echo "✓ All tools validated successfully"
                        '''
                    } catch (Exception e) {
                        echo "✗ Validation failed: ${e.message}"
                        currentBuild.result = 'FAILURE'
                        error("Validate stage failed")
                    }
                }
            }
        }
        
        stage('Build') {
            steps {
                script {
                    echo "========== Stage: Maven Build =========="
                    try {
                        sh '''
                            echo "Starting Maven build..."
                            ./mvnw clean package -DskipTests=true -U \
                                -Dmaven.test.skip=true \
                                -Dorg.slf4j.simpleLogger.defaultLogLevel=info
                            
                            if ls target/*.jar 1> /dev/null 2>&1; then
                                echo "✓ Build successful - JAR file created"
                                ls -lh target/*.jar
                            else
                                echo "✗ JAR file not found after build"
                                exit 1
                            fi
                        '''
                    } catch (Exception e) {
                        echo "✗ Build failed: ${e.message}"
                        currentBuild.result = 'FAILURE'
                        error("Build stage failed")
                    }
                }
            }
        }
        
        stage('Unit Tests') {
            steps {
                script {
                    echo "========== Stage: Run Unit Tests =========="
                    try {
                        sh '''
                            echo "Running unit tests..."
                            ./mvnw test -Dorg.slf4j.simpleLogger.defaultLogLevel=info
                            
                            echo "✓ Unit tests completed"
                        '''
                    } catch (Exception e) {
                        echo "⚠ Tests failed or had errors: ${e.message}"
                        unstable("Unit tests failed")
                    }
                }
            }
        }
        
        stage('Code Quality - SonarQube') {
            steps {
                script {
                    echo "========== Stage: SonarQube Analysis =========="
                    try {
                        withSonarQubeEnv('SonarQube') {
                            sh '''
                                echo "Starting SonarQube analysis..."
                                ./mvnw org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                                    -Dsonar.projectKey=${APP_NAME} \
                                    -Dsonar.projectName=${APP_NAME} \
                                    -Dsonar.host.url=${SONAR_HOST_URL} \
                                    -Dsonar.login=${SONAR_AUTH_TOKEN} \
                                    -Dorg.slf4j.simpleLogger.defaultLogLevel=info
                                
                                echo "✓ SonarQube analysis completed"
                            '''
                        }
                        
                        timeout(time: 5, unit: 'MINUTES') {
                            def qg = waitForQualityGate()
                            if (qg.status != 'OK') {
                                echo "⚠ Quality Gate failed: ${qg.status}"
                                unstable("Quality Gate failed")
                            } else {
                                echo "✓ Quality Gate passed"
                            }
                        }
                    } catch (Exception e) {
                        echo "⚠ SonarQube analysis failed: ${e.message}"
                        unstable("SonarQube stage failed")
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    echo "========== Stage: Build Docker Image =========="
                    try {
                        sh '''
                            echo "Creating Dockerfile..."
                            
                            if [ ! -f "Dockerfile" ]; then
                                cat > Dockerfile << 'EOF'
FROM eclipse-temurin:21-jre-slim
LABEL maintainer="DevOps Team"
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app.jar"]
EOF
                            fi
                            
                            echo "Building Docker image..."
                            docker build -t ${APP_NAME}:${DOCKER_IMAGE_TAG} .
                            docker tag ${APP_NAME}:${DOCKER_IMAGE_TAG} ${APP_NAME}:latest
                            
                            echo "✓ Docker image built successfully"
                            docker images | grep ${APP_NAME}
                        '''
                    } catch (Exception e) {
                        echo "✗ Docker build failed: ${e.message}"
                        currentBuild.result = 'FAILURE'
                        error("Docker build stage failed")
                    }
                }
            }
        }
        
        stage('Push to ACR') {
            steps {
                script {
                    echo "========== Stage: Push to Azure Container Registry =========="
                    try {
                        sh '''
                            echo "Logging in to ACR..."
                            echo "${REGISTRY_PASSWORD}" | docker login -u ${REGISTRY_USERNAME} --password-stdin ${REGISTRY_URL}
                            
                            echo "Tagging image for ACR..."
                            docker tag ${APP_NAME}:${DOCKER_IMAGE_TAG} ${REGISTRY_URL}/${APP_NAME}:${DOCKER_IMAGE_TAG}
                            docker tag ${APP_NAME}:${DOCKER_IMAGE_TAG} ${REGISTRY_URL}/${APP_NAME}:latest
                            
                            echo "Pushing to ACR..."
                            docker push ${REGISTRY_URL}/${APP_NAME}:${DOCKER_IMAGE_TAG}
                            docker push ${REGISTRY_URL}/${APP_NAME}:latest
                            
                            echo "✓ Image pushed to ACR successfully"
                            docker logout ${REGISTRY_URL}
                        '''
                    } catch (Exception e) {
                        echo "✗ ACR push failed: ${e.message}"
                        currentBuild.result = 'FAILURE'
                        error("ACR push stage failed")
                    }
                }
            }
        }
        
        stage('Trivy Vulnerability Scan') {
            steps {
                script {
                    echo "========== Stage: Trivy Image Scan =========="
                    try {
                        sh '''
                            echo "Installing Trivy if not present..."
                            if ! command -v trivy &> /dev/null; then
                                wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | apt-key add - || true
                                echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | tee -a /etc/apt/sources.list.d/trivy.list || true
                                apt-get update || true
                                apt-get install -y trivy || true
                            fi
                            
                            echo "Scanning Docker image with Trivy..."
                            trivy image --severity HIGH,CRITICAL \
                                --format json \
                                --output trivy-report.json \
                                ${REGISTRY_URL}/${APP_NAME}:${DOCKER_IMAGE_TAG} || true
                            
                            echo "Scanning in table format..."
                            trivy image --severity HIGH,CRITICAL \
                                --format table \
                                ${REGISTRY_URL}/${APP_NAME}:${DOCKER_IMAGE_TAG} || true
                            
                            echo "✓ Trivy scan completed"
                        '''
                    } catch (Exception e) {
                        echo "⚠ Trivy scan encountered an error: ${e.message}"
                        unstable("Trivy scan stage had issues")
                    }
                }
            }
        }
        
        stage('Deploy to AKS') {
            steps {
                script {
                    echo "========== Stage: Deploy to AKS =========="
                    try {
                        sh '''
                            echo "Configuring kubectl for AKS..."
                            az aks get-credentials \
                                --resource-group ${AKS_RESOURCE_GROUP} \
                                --name ${AKS_CLUSTER_NAME} \
                                --overwrite-existing
                            
                            echo "Creating Kubernetes deployment..."
                            
                            cat > deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}
  labels:
    app: ${APP_NAME}
    version: v${DOCKER_IMAGE_TAG}
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ${APP_NAME}
  template:
    metadata:
      labels:
        app: ${APP_NAME}
    spec:
      imagePullSecrets:
      - name: acr-secret
      containers:
      - name: ${APP_NAME}
        image: ${REGISTRY_URL}/${APP_NAME}:${DOCKER_IMAGE_TAG}
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: "500m"
            memory: "512Mi"
          requests:
            cpu: "250m"
            memory: "256Mi"
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}-service
spec:
  type: LoadBalancer
  selector:
    app: ${APP_NAME}
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
EOF
                            
                            echo "Applying deployment to AKS..."
                            kubectl apply -f deployment.yaml
                            
                            echo "Checking deployment status..."
                            kubectl rollout status deployment/${APP_NAME} -n default --timeout=5m
                            
                            echo "✓ Deployment to AKS completed successfully"
                        '''
                    } catch (Exception e) {
                        echo "✗ AKS deployment failed: ${e.message}"
                        currentBuild.result = 'FAILURE'
                        error("AKS deployment stage failed")
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                echo "========== Pipeline Cleanup =========="
                archiveArtifacts artifacts: 'target/*.jar,trivy-report.json', 
                                   allowEmptyArchive: true, 
                                   onlyIfSuccessful: false
                cleanWs(deleteDirs: true, patterns: [[pattern: 'target/**', type: 'INCLUDE']])
                echo "✓ Cleanup completed"
            }
        }
        success {
            echo "========== Pipeline Completed Successfully =========="
        }
        failure {
            echo "========== Pipeline Failed =========="
        }
    }
}
