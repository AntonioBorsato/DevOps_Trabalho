pipeline {
    agent any

    stages {
        stage('Git Pull & Build Containers') {
            steps {
                script {
                    git branch: "main", url: "https://github.com/AntonioBorsato/DevOps_Trabalho.git"
                    sh 'docker-compose down -v || true' // Ignorar erros se os contêineres não estiverem ativos
                    sh 'docker-compose build'
                }
            }
        }

        stage('Start Containers & Run Tests') {
            steps {
                script {
                    sh 'docker-compose up -d mariadb flask test mysqld_exporter prometheus grafana'
                    // Esperar ou verificar a disponibilidade dos serviços
                    sh 'for i in {1..40}; do sleep 1; done'

                    catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                        sh 'docker-compose run --rm test'
                    }
                }
            }
        }

        stage('Keep Application Running') {
            steps {
                script {
                    // Reiniciar sem o contêiner de teste
                    sh 'docker-compose up -d mariadb flask mysqld_exporter prometheus grafana'
                }
            }
        }
    }

    post {
        always {
            script {
                echo 'Pipeline concluída. Parando todos os contêineres.'
                sh 'docker-compose down -v'
            }
        }
    }
}
