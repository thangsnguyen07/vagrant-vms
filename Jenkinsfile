def COLOR_MAP = [
    'SUCCESS': 'good', 
    'FAILURE': 'danger',
]

pipeline{
    agent any
    
    tools {
        maven "MAVEN3.9"
        jdk "JDK17"
    }

    environment {
        registryCredential = 'ecr:ap-southeast-1:awscreds'
        imageName = '058264494511.dkr.ecr.ap-southeast-1.amazonaws.com/vprofileappimg'
        vprofileRegistry = 'https://058264494511.dkr.ecr.ap-southeast-1.amazonaws.com'
    }

    stages {
        stage('FETCH CODE') {
            steps {
                git branch: 'docker', url: 'https://github.com/hkhcoder/vprofile-project.git'
            }
        }

        stage('BUILD') {
            steps {
                sh 'mvn install -DskipTests'
            }

            post {
                success {
                    echo "Archiving artifact"
                    archiveArtifacts artifacts: "**/*.war"
                }
            }
        }

        stage('UNIT TEST') {
            steps {
                sh 'mvn test'
            }
        }

        stage('CHECKSTYPE ANALYSIS') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
        }

        stage('SonarQube Code Analysis') {
            environment {
                scannerHome = tool 'sonar6.2'
            }
            steps {
                withSonarQubeEnv('sonarserver') {
                    sh '''
                        ${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                        -Dsonar.projectName=vprofile \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=src/ \
                        -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                        -Dsonar.junit.reportsPath=target/surefire-reports/ \
                        -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                        -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml
                    '''
                }
            }
        }

        stage("Quality Gate") {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    // Parameter indicates whether to set pipeline to UNSTABLE if Quality Gate fails
                    // true = set pipeline to UNSTABLE, false = don't
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build App Image') {
            steps {
                script {
                    dockerImage = docker.build(imageName + ":$BUILD_NUMBER", "./Docker-files/app/multistage/")
                }
            }
        }

        stage('Upload App Image') {
            steps {
                script {
                    docker.withRegistry(vprofileRegistry, registryCredential) {
                        dockerImage.push("$BUILD_NUMBER")
                        dockerImage.push('latest')
                    }
                }
            }

            post {
                always {
                    echo 'Cleaning up...'
                    sh 'docker rmi -f $(docker images -a -q)'
                }
            }
        }

        // stage('Upload Artifact') {
        //     steps {
        //         nexusArtifactUploader(
        //             nexusVersion: 'nexus3',
        //             protocol: 'http',
        //             nexusUrl: '192.168.33.11:8081',
        //             groupId: 'QA',
        //             version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
        //             repository: 'vprofile-repo',
        //             credentialsId: 'nexuslogin',
        //             artifacts: [
        //                 [artifactId: 'vproapp',
        //                 classifier: '',
        //                 file: 'target/vprofile-v2.war',
        //                 type: 'war']
        //             ]
        //         )
        //     }
        // }
    }

    post {
        always {
            echo 'Slack Notification.'
            slackSend channel: '#devops',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
        }
    }
}