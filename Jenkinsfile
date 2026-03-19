pipeline {
    agent none

    stages {
        stage('worker') {
            stages {
                stage('worker-build') {
                    when {
                        changeset '**/worker/**'
                    }
                    agent {
                        docker {
                            image 'maven:3.9.8-sapmachine-21'
                            args '-v $HOME/.m2:/root/.m2'
                        }
                    }
                    steps {
                        echo 'Compiling worker app'
                        dir('worker') {
                            sh 'mvn compile'
                        }
                    }
                }
                stage('worker-test') {
                    when {
                        changeset '**/worker/**'
                    }

                    agent {
                        docker {
                            image 'maven:3.9.8-sapmachine-21'
                            args '-v $HOME/.m2:/root/.m2'
                        }
                    }

                    steps {
                        echo 'Running Unit Tests on worker app'
                        dir('worker') {
                            sh 'mvn clean test'
                        }
                    }
                }
                stage('worker-package') {
                    when {
                        branch 'master'
                        changeset '**/worker/**'
                    }

                    agent {
                        docker {
                            image 'maven:3.9.8-sapmachine-21'
                            args '-v $HOME/.m2:/root/.m2'
                        }
                    }

                    steps {
                        echo 'Packaging worker app with Maven and archiving artifacts.'
                        dir('worker') {
                            sh 'mvn package -DskipTests'
                            archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
                        }
                    }
                }
                stage('worker-docker-package') {
                    agent any
                    when {
                        changeset '**/worker/**'
                        branch 'master'
                    }
                    steps {
                        echo 'Packaging worker app with docker'
                        script {
                            docker.withRegistry('https://index.docker.io/v1/', 'dockerlogin') {
                                def workerImage = docker.build("sicness9/worker:v${env.BUILD_ID}", './worker')
                                workerImage.push()
                                workerImage.push('latest')
                            }
                        }
                    }
                }
            }
        }
        stage('vote') {
            stages {
                stage('vote-build') {
                    agent {
                        docker {
                            image 'python:3.11-slim'
                            args '--user root'
                        }
                    }
                    when {
                        changeset '**/vote/**'
                    }

                    steps {
                        echo 'Compiling vote app.'
                        dir('vote') {
                            sh 'pip install -r requirements.txt'
                        }
                    }
                }
                stage('vote-test') {
                    agent {
                        docker {
                            image 'python:3.11-slim'
                            args '--user root'
                        }
                    }
                    when {
                        changeset '**/vote/**'
                    }

                    steps {
                        echo 'Running Unit Tests on vote app.'
                        dir('vote') {
                            sh 'pip install -r requirements.txt'
                            sh 'nosetests -v'
                        }
                    }
                }
                stage('vote-integration-test') {
                    agent any
                    when {
                        changeset '**/vote/**'
                        branch 'master'
                    }

                    steps {
                        echo 'Running Integration Tests on vote app'

                        dir('vote') {
                            sh 'sh integration_test.sh'
                        }
                    }
                }
                stage('vote-docker-package') {
                    agent any
                    when {
                        changeset '**/vote/**'
                        branch 'master'
                    }

                    steps {
                        echo 'Packaging vote app with docker'
                        script {
                            docker.withRegistry('https://index.docker.io/v1/', 'dockerlogin') {
                                // ./vote is the path to the Dockerfile that Jenkins will find from the Github repo
                                def voteImage = docker.build("sicness9/vote:v${env.BUILD_ID}", './vote')
                                voteImage.push()
                                voteImage.push('latest')
                            }
                        }
                    }
                }
            }
        }
        stage('result') {
            agent {
                docker {
                    image 'node:22.4.0'
                }
            }
            stages {
                stage('result-build') {
                    when {
                        changeset '**/result/**'
                    }

                    steps {
                        echo 'Building result app'
                        dir('result') {
                            sh 'npm install'
                        }
                    }
                }
                stage('result-test') {
                    when {
                        changeset '**/result/**'
                    }

                    steps {
                        echo 'Running Unit Tests on result app'
                        dir('result') {
                            sh 'npm install'
                            sh 'npm test'
                        }
                    }
                }
                stage('result-docker-package') {
                    agent any
                    when {
                        changeset '**/result/**'
                        branch 'master'
                    }

                    steps {
                        echo 'Packaging result app with docker'
                        script {
                            docker.withRegistry('https://index.docker.io/v1/', 'dockerlogin') {
                                def resultImage = docker.build("sicness9/result:v${env.BUILD_ID}", '/.vote')
                                resultImage.push()
                                resultImage.push('latest')
                            }
                        }
                    }
                }
            }
        }
        stage('Sonarqube') {
            agent any
            when {
                branch 'master'
            }

            environment {
                sonarpath = tool 'SonarScanner'
            }

            steps {
                echo 'Running Sonarqube Analysis..'
                withSonarQubeEnv('sonar-instavote') {
                    sh "${sonarpath}/bin/sonar-scanner -Dproject.settings=sonar-project.properties -Dorg.jenkinsci.plugins.durabletask.BourneShellScript.HEARTBEAT_CHECK_INTERVAL=86400"
                }
            }
        }
        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    // Parameter indicates whether to set pipeline to UNSTABLE if quality gate failes
                    // true = set pipeline to unstable, false = don't
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('deploy to dev') {
            agent any
            steps {
                echo 'Deploy instavote app with docker compose'
                sh 'docker compose up -d'
            }
        }
    }

    post {
        always {
            echo 'Building multibranch pipeline job for worker is completed..'
        }
    }
}
