pipeline {
    agent none

    stages {
        stage('worker') {
            stages {
                stage('build') {
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
                stage('test') {
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
                stage('package') {
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
                stage('docker-package') {
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
                                workerImage.push("${env.BRANCH_NAME}")
                                workerImage.push('latest')
                            }
                        }
                    }
                }
            }
        }
        stage('vote') {
            stages {
                stage('build') {
                    agent {
                        docker {
                            image 'python:3.11-slim'
                            args '--user root'
                        }
                    }

                    steps {
                        echo 'Compiling vote app.'
                        dir('vote') {
                            sh 'pip install -r requirements.txt'
                        }
                    }
                }
                stage('test') {
                    agent {
                        docker {
                            image 'python:3.11-slim'
                            args '--user root'
                        }
                    }
                    steps {
                        echo 'Running Unit Tests on vote app.'
                        dir('vote') {
                            sh 'pip install -r requirements.txt'
                            sh 'nosetests -v'
                        }
                    }
                }

                stage('vote-docker-package') {
                    agent any

                    steps {
                        echo 'Packaging vote app with docker'
                        script {
                            docker.withRegistry('https://index.docker.io/v1/', 'dockerlogin') {
                                // ./vote is the path to the Dockerfile that Jenkins will find from the Github repo
                                def voteImage = docker.build("xxxx/vote:v${env.BUILD_ID}", './vote')
                                voteImage.push()
                                voteImage.push("${env.BRANCH_NAME}")
                                voteImage.push('latest')
                            }
                        }
                    }
                }
            }
        }
        stage('result') {
            stages {
                agent {
                    docker {
                        image 'node:22.4.0'
                    }
                }

                stages {
                    stage('build') {
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
                    stage('test') {
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
                }
            }
        }
    }

    post {
        always {
            echo 'Building multibranch pipeline job for worker is completed..'
        }
    }
}
