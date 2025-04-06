pipeline {
    agent any

    tools {
        maven 'Maven-3.9.9'
    }

    options {
        skipDefaultCheckout()
    }

    environment {
        BUILD_VETS = "false"
        BUILD_VISITS = "false"
        BUILD_CUSTOMERS = "false"
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    checkout scm
                }
            }
        }

        stage('Detect Changes') {
            steps {
                script {
                    def baseCommit = sh(script: '''
                        git fetch origin main || true
                        if git show-ref --verify --quiet refs/remotes/origin/main; then
                            git merge-base origin/main HEAD
                        else
                            git rev-parse HEAD~1
                        fi
                    ''', returnStdout: true).trim()

                    def changedServices = sh(
                        script: "git diff --name-only ${baseCommit} HEAD | awk -F/ '{print \$1}' | sort -u",
                        returnStdout: true
                    ).trim().split('\n')

                    def allServices = [
                        'spring-petclinic-vets-service',
                        'spring-petclinic-visits-service',
                        'spring-petclinic-customers-service',
                        'spring-petclinic-genai-service'
                    ]

                    def changedServicesList = changedServices as List
                    env.SERVICES_TO_BUILD = allServices.findAll { it in changedServicesList }.join(',')
                    echo "Services to build: ${env.SERVICES_TO_BUILD}"

                    // Đặt giá trị true/false cho từng service
                    env.BUILD_VETS = changedServicesList.contains("spring-petclinic-vets-service").toString()
                    env.BUILD_VISITS = changedServicesList.contains("spring-petclinic-visits-service").toString()
                    env.BUILD_CUSTOMERS = changedServicesList.contains("spring-petclinic-customers-service").toString()
                }
            }
        }

        stage('Build & Test Services') {
            matrix {
                axes {
                    axis {
                        name 'SERVICE'
                        values 'spring-petclinic-vets-service',
                               'spring-petclinic-visits-service',
                               'spring-petclinic-customers-service'
                    }
                }

                when {
                    expression {
                        return (SERVICE == 'spring-petclinic-vets-service' && env.BUILD_VETS == "true") ||
                               (SERVICE == 'spring-petclinic-visits-service' && env.BUILD_VISITS == "true") ||
                               (SERVICE == 'spring-petclinic-customers-service' && env.BUILD_CUSTOMERS == "true")
                    }
                }

                stages {
                    stage('Build') {
                        steps {
                            dir("${SERVICE}") {
                                sh "mvn clean package -DskipTests"
                            }
                        }
                    }
                    stage('Test') {
                        steps {
                            dir("${SERVICE}") {
                                sh "mvn test verify"
                                junit '**/target/surefire-reports/*.xml'
                            }
                        }
                    }
                }

            }
        }
        stage('Publish Coverage') {
                    steps {
                        jacoco(
                            execPattern: '**/target/jacoco.exec',
                            classPattern: '**/target/classes',
                            sourcePattern: '**/src/main/java',
                            inclusionPattern: '**/*.class',
                            exclusionPattern: '**/*Test.class',
                            minimumInstructionCoverage: '70',
                            minimumBranchCoverage: '70'
                        )
                    }
                }
    }

    post {
        always {
            echo "Pipeline completed!"
        }
    }
}
