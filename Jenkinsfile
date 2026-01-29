@Library('Shared') _
pipeline {
    agent any
    tools {
    nodejs 'nodejs'
}
    
    environment{
        SONAR_HOME = tool "Sonar"
    }
    
    parameters {
        string(name: 'FRONTEND_DOCKER_TAG', defaultValue: '', description: 'Setting docker image for latest push')
        string(name: 'BACKEND_DOCKER_TAG', defaultValue: '', description: 'Setting docker image for latest push')
    }
    
    stages {
        stage("Validate Parameters") {
            steps {
                script {
                    if (params.FRONTEND_DOCKER_TAG == '' || params.BACKEND_DOCKER_TAG == '') {
                        error("FRONTEND_DOCKER_TAG and BACKEND_DOCKER_TAG must be provided.")
                    }
                }
            }
        }
        stage("Workspace cleanup"){
            steps{
                script{
                    cleanWs()
                }
            }
        }
        
        stage('Git: Code Checkout') {
            steps {
                script{
                    checkout scm
                }
            }
        }
        
        stage("Trivy: Filesystem scan"){
            steps{
                script{
                    trivy_scan()
                }
            }
        }
stage('Install Dependencies') {
    parallel {
        stage('Frontend deps') {
            steps {
                dir('frontend') {
                    sh 'npm install'
                    sh 'npm audit fix'
                }
            }
        }
        stage('Backend deps') {
            steps {
                dir('backend') {
                    sh 'npm install'
                    sh 'npm audit fix'
                }
            }
        }
    }
}



        stage("OWASP: Dependency check") {
            steps {
                withCredentials([string(credentialsId: 'NVD_API_KEY', variable: 'NVD_API_KEY')]) {
                    owasp_dependency(
                        additionalArguments: "--nvdApiKey ${NVD_API_KEY}"
                    )
                }
            }
        }

        
        stage("SonarQube: Code Analysis"){
            steps{
                script{
                    sonarqube_analysis("Sonar","wanderlust","wanderlust")
                }
            }
        }
        
        stage("SonarQube: Code Quality Gates"){
            steps{
                script{
                    sonarqube_code_quality()
                }
            }
        }
        
        stage('Exporting environment variables') {
            parallel{
                stage("Backend env setup"){
                    steps {
                        script{
                            dir("Automations"){
                                sh "bash updatebackendnew.sh"
                            }
                        }
                    }
                }
                
                stage("Frontend env setup"){
                    steps {
                        script{
                            dir("Automations"){
                                sh "bash updatefrontendnew.sh"
                            }
                        }
                    }
                }
            }
        }
        
        stage("Docker: Build Images"){
            steps{
                script{
                        dir('backend'){
                            docker_build("wanderlust-backend-beta","${params.BACKEND_DOCKER_TAG}","de2pressed")
                        }
                    
                        dir('frontend'){
                            docker_build("wanderlust-frontend-beta","${params.FRONTEND_DOCKER_TAG}","de2pressed")
                        }
                }
            }
        }
        
        stage("Docker: Push to DockerHub"){
            steps{
                script{
                    docker_push("wanderlust-backend-beta","${params.BACKEND_DOCKER_TAG}","de2pressed") 
                    docker_push("wanderlust-frontend-beta","${params.FRONTEND_DOCKER_TAG}","de2pressed")
                }
            }
        }
    }
    post{
        success{
            archiveArtifacts artifacts: '*.xml', followSymlinks: false
            build job: "Wanderlust-CD", parameters: [
                string(name: 'FRONTEND_DOCKER_TAG', value: "${params.FRONTEND_DOCKER_TAG}"),
                string(name: 'BACKEND_DOCKER_TAG', value: "${params.BACKEND_DOCKER_TAG}")
            ]
        }
    }
}
