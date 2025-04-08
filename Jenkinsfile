pipeline {
    agent any
    options {
        skipDefaultCheckout()
    }
    environment {
        MAVEN_OPTS = "Maven-3.9.9"
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Determine Changed Services') {
            steps {
                script {
                    def changedServices = sh(script: '''
                        git fetch origin main
                        git diff --name-only origin/main...HEAD | awk -F/ '{print $1}' | sort -u
                    ''', returnStdout: true).trim().split('\n')

                    def allServices = [
                        'spring-petclinic-vets-service',
                        'spring-petclinic-visits-service',
                        'spring-petclinic-customers-service',
                        'spring-petclinic-genai-service'
                    ]

                    def changedServicesList = changedServices as List
                    env.SERVICES_TO_BUILD = allServices.findAll { it in changedServicesList }.join(',')
                }
            }
        }
        stage('Test') {
            when {
                expression { return env.SERVICES_TO_BUILD?.trim() }
            }
            steps {
                script {
                    def services = env.SERVICES_TO_BUILD.split(',')
                    for (s in services) {
                        dir("${s}") {
                            echo "Testing service: ${s}"
                            sh "mvn clean test"
                            junit '**/target/surefire-reports/*.xml'
                            jacoco execPattern: '**/target/jacoco.exec', classPattern: '**/target/classes', sourcePattern: '**/src/main/java'

                            // Tính toán code coverage
                            def missed = sh(script: "grep -oPm1 '(?<=<counter type=\"INSTRUCTION\" missed=\")[0-9]+' target/site/jacoco/jacoco.xml", returnStdout: true).trim().toInteger()
                            def covered = sh(script: "grep -oPm1 '(?<=<counter type=\"INSTRUCTION\" covered=\")[0-9]+' target/site/jacoco/jacoco.xml", returnStdout: true).trim().toInteger()
                            def total = missed + covered
                            def coveragePercent = (100 * covered) / total

                            echo "${s} Coverage: ${coveragePercent}%"

                            if (coveragePercent < 70) {
                                error "${s} code coverage is below 70% (${coveragePercent}%). Failing pipeline."
                            }
                        }
                    }
                }
            }
        }
        stage('Build') {
            when {
                expression { return env.SERVICES_TO_BUILD?.trim() }
            }
            steps {
                script {
                    def services = env.SERVICES_TO_BUILD.split(',')
                    for (s in services) {
                        dir("${s}") {
                            echo "Building service: ${s}"
                            sh "mvn clean package"
                        }
                    }
                }
            }
        }
    }
    post {
        always {
            echo "Pipeline complete"
        }
    }
}
