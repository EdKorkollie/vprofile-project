def COLOR_MAP = [
    'SUCCESS': 'good',  //good in slack means green color
    'FAILURE': 'danger' // danger in slack means red color
]
pipeline {
    agent any
    tools {
        maven "MAVEN3" //given the same name you have given in jenkins
        jdk "OracleJDK8"
    }

    //Defining all of our variables that are used in settings.xml and pom.xml file
    environment {
        SNAP_REPO = 'vprofile-snapshot' // snapshot repo that was created in nexus
        NEXUS_USER = 'admin'
        NEXUS_PASS = 'admin123'
        RELEASE_REPO = 'vprofile-release' //ewpo CREATE IN NEXUS
        CENTRAL_REPO = 'vpro-maven-central'
        NEXUSIP = 'private nexus server IP'
        NEXUSPORT = '8081'
        NEXUS_GRP_REPO = 'vpro-maven-group' //group repo created in nexus
        NEXUS_LOGIN = 'nexuslogin' //nexus login created in jenkins
        SONARSERVER = 'sonarserver' //storing sonar server name as an env variable. this is the name we saved in jenkins
        SONARSCANNER = 'sonarscanner' //storing sonar scanner name as an env variable. this is the name we saved in jenkins
    }

    stages {
        stage('Build') {
            steps {
                sh 'mvn -s settings.xml -DskipTests install' // pass some settings and skip unit test
            }
            post {
                success {
                    echo "Now Archiving."
                    archiveArtifacts artifacts: '**/*.war'  //this means archive everything that ends with .war
                }
            }
        }

        stage('Test') {
            steps {
                sh 'mvn -s settings.xml test'
            }
        }

        stage('Checkstyle Analysis') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
        }

        stage('Sonar Anaylsis') {
            environment {
                scannerHome = tool "${SONARSCANNER}"
            }
            steps {
                withSonarQubeEnv("${SONARSERVER}") {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile-repo \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    // timeout is so that it does not wait for infinity. you can change to minutes
                    //parameter indicates wheter to set pipeline to unstable
                    // true = set pipeline to UNSTABLE, false = dont 
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('UploadArtifact') {
            steps {
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: "${NEXUSIP}:${NEXUSPORT}",
                    groupId: 'QA',
                    version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}"
                    repository: "${RELEASE_REPO}",
                    credentialsId: "${NEXUS_LOGIN}",
                    artifacts: [
                        [artifactId: 'vproapp',
                         classifier: '',
                         file: 'target/vprofile-v2.war',
                         type: 'war']
                    ]
                )
            }
        }
    }
    post {

        always {
            echo 'Slack Notification.'
            slackSend channel: '#jenkinscicd',
            color: COLOR_MAP[currentBuild.currentResult],
            message: "*${currentBuild.currentReslt}:* Job ${end.JOB_NAME} build ${end.BUILD_Number} \n more info at: ${env.BUILD_URL}"
        }
    }
}