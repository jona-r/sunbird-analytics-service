pipeline {
    agent any
    environment {
        hub_org = 'registery/sunbirded'
    }
    stages {
        stage('Checkout') {
            steps {
                script {
                    if (!env.hub_org) {
                        echo "\u001B[1m\u001B[31mUh Oh! Please set a Jenkins environment variable named hub_org with value as registery/sunbirded\u001B[0m"
                        error 'Please resolve the errors and rerun..'
                    } else {
                        echo "\u001B[1m\u001B[32mFound environment variable named hub_org with value as: ${env.hub_org}\u001B[0m"
                    }
                }
                cleanWs()
                checkout scm
                script {
                    commit_hash = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    build_tag = sh(script: "echo ${params.github_release_tag.split('/')[-1]}_${commit_hash}_${env.BUILD_NUMBER}", returnStdout: true).trim()
                    echo "build_tag: ${build_tag}"
                }
            }
        }
        stage('Build') {
            steps {
                script {
                    env.NODE_ENV = "build"
                    echo "Environment will be : ${env.NODE_ENV}"
                    sh '''
                        export JAVA_HOME=/usr/lib/jvm/jdk-11.0.2
                        export PATH=$JAVA_HOME/bin:$PATH
                        java -version
                        mvn clean install -DskipTests
                        mvn play2:dist -pl analytics-api
                    '''
                }
            }
        }
        stage('Package') {
            steps {
                dir('sunbird-analytics-service-distribution') {
                    sh """
                       cp ../analytics-api/target/analytics-api-2.0-dist.zip .
                       /opt/apache-maven-3.6.3/bin/mvn package -Pbuild-docker-image -Drelease-version=${build_tag}
                    """
                }
            }
        }
        stage('Retagging') {
            steps {
                sh """
                    docker tag sunbird-analytics-service:${build_tag} ${env.hub_org}/sunbird-analytics-service:${build_tag}
                    echo {\\"image_name\\" : \\"sunbird-analytics-service\\", \\"image_tag\\" : \\"${build_tag}\\", \\"node_name\\" : \\"${env.NODE_NAME}\\"} > metadata.json
                """
            }
        }
        stage('ArchiveArtifacts') {
            steps {
                archiveArtifacts artifacts: "metadata.json", allowEmptyArchive: true
                script {
                    currentBuild.description = "${build_tag}"
                }
            }
        }
    }
    post {
        failure {
            script {
                currentBuild.result = "FAILURE"
            }
        }
    }
}
