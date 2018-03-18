pipeline {
    tools {
        maven 'Maven 3'
        jdk 'OracleJDK 8'
    }

    agent any

    options {
        timestamps()
        timeout(time: 5, unit: 'MINUTES')
    }

    triggers {
        githubPush()
    }

    stages {
        stage ('check-commit') {
            steps {
                script {
                    env.CI_SKIP = "false"
                    result = sh (script: "git log -1 | grep '(?s).[CI[-\\s]SKIP].*'", returnStatus: true)
                    if (result == 0) {
                        env.CI_SKIP = "true"
                        error "'[CI-SKIP]' found in git commit message. Aborting."
                    }
                }
            }
        }
        stage ('compile') {
            steps {
                withCredentials([string(credentialsId: 'authme-coveralls-token', variable: 'COVERALLS_TOKEN')]) {
                    sh 'mvn clean verify jacoco:report coveralls:report -DrepoToken=${COVERALLS_TOKEN}'
                }
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                    jacoco(execPattern: '**/**.exec', classPattern: '**/classes', sourcePattern: '**/src/main/java')
                }
            }
        }
        stage ('sources') {
            when {
                branch "master"
            }
            steps {
                sh 'mvn source:jar'
            }
            post {
                success {
                    archiveArtifacts artifacts: 'target/*-souces.jar', fingerprint: true
                }
            }
        }
        stage ('javadoc') {
            when {
                branch "master"
            }
            steps {
                sh 'mvn javadoc:javadoc javadoc:jar'
            }
            post {
                success {
                    step([
                        $class: 'JavadocArchiver',
                        javadocDir: 'target/site/apidocs',
                        keepAll: true
                    ])
                    archiveArtifacts artifacts: 'target/*-javadoc.jar', fingerprint: true
                }
            }
        }
        stage ('deploy') {
            when {
                branch "master"
            }
            steps {
                sh 'mvn -DskipTests deploy'
            }
        }
    }

    post {
        always {
            script {
                if (env.CI_SKIP == "true") {
                    currentBuild.result = 'NOT_BUILT'
                }
            }
            withCredentials([string(credentialsId: 'authme-discord-webhook', variable: 'DISCORD_WEBHOOK')]) {
                discordSend webhookURL: '${DISCORD_WEBHOOK}'
            }
        }
        success {
            githubNotify description: 'The jenkins build was successful',  status: 'SUCCESS'
        }
        failure {
            githubNotify description: 'The jenkins build failed',  status: 'FAILURE'
        }
    }
}
