pipeline {
    agent any

    parameters {
        booleanParam(
            name: 'PUSH_DOCKER_HUB',
            defaultValue: false,
            description: 'Opcional: publicar a imagem no Docker Hub após o build local.'
        )
    }

    environment {
        DOCKER_IMAGE = 'andprof/jogo-enigma-api'
        DOCKER_CREDENTIALS_ID = 'dockerhub-credentials'
        IMAGE_TAG = "${BUILD_NUMBER}"
        COMPOSE_FILE = 'docker-compose.homol.yml'
    }

    stages {
        stage('1 - Checkout') {
            steps {
                checkout scm
            }
        }

        stage('2 - CI - Testes JUnit + JaCoCo') {
            steps {
                bat './mvnw clean test'
            }
            post {
                always {
                    junit testResults: 'target/surefire-reports/*.xml', allowEmptyResults: false
                    archiveArtifacts artifacts: 'target/site/jacoco/**', allowEmptyArchive: true
                }
            }
        }

        stage('3 - CI - Qualidade PMD') {
            steps {
                bat 'mvnw.cmd pmd:pmd -DskipTests'
            }
            post {
                always {
                    archiveArtifacts artifacts: 'target/pmd.html', allowEmptyArchive: true
                }
            }
        }

        stage('4 - CI - Package') {
            steps {
                bat 'mvnw.cmd package -DskipTests'
            }
        }

        stage('5 - CD - Build dos Containers') {
            steps {
                bat '''
                    set DOCKER_IMAGE=%DOCKER_IMAGE%
                    set IMAGE_TAG=%IMAGE_TAG%
                    docker compose -f %COMPOSE_FILE% build api bff
                '''
            }
        }

        stage('6 - Registry - Docker Hub (opcional)') {
            when {
                expression { return params.PUSH_DOCKER_HUB }
            }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: env.DOCKER_CREDENTIALS_ID,
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    bat '''
                        echo %DOCKER_PASSWORD% | docker login -u %DOCKER_USER% --password-stdin
                        docker push %DOCKER_IMAGE%:%IMAGE_TAG%
                        docker logout
                    '''
                }
            }
        }

        stage('7 - CD - Deploy HOMOL') {
            steps {
                bat '''
                    set DOCKER_IMAGE=%DOCKER_IMAGE%
                    set IMAGE_TAG=%IMAGE_TAG%
                    docker compose -f %COMPOSE_FILE% down --remove-orphans
                    docker compose -f %COMPOSE_FILE% up -d
                '''
            }
        }

        stage('8 - CD - Health Check HOMOL') {
            steps {
                powershell '''
                    $ok = $false
                    for ($i = 1; $i -le 12; $i++) {
                        try {
                            $r = Invoke-RestMethod -Uri "http://localhost:8080/actuator/health" -TimeoutSec 5
                            if ($r.status -eq "UP") { $ok = $true; break }
                        } catch {
                            Write-Host "API ainda nao pronta. Tentativa $i/12..."
                        }
                        Start-Sleep -Seconds 5
                    }
                    if (-not $ok) { throw "Health check da API falhou." }
                '''
            }
        }

        stage('9 - CD - Cypress E2E') {
            steps {
                dir('frontend') {
                    bat '''
                        npm install
                        set CYPRESS_BASE_URL=http://localhost:3000
                        npm run test:e2e
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'frontend/cypress/screenshots/**,frontend/cypress/videos/**', allowEmptyArchive: true
                }
            }
        }

        stage('10 - Observabilidade') {
            steps {
                echo 'API: http://localhost:8080'
                echo 'BFF/Front: http://localhost:3000'
                echo 'Prometheus: http://localhost:9091'
                echo 'Grafana: http://localhost:3001'
            }
        }
    }

    post {
        success {
            echo 'Pipeline CI/CD concluido: build, testes, container, deploy HOMOL e E2E aprovados.'
        }
        failure {
            echo 'Pipeline interrompido. Identifique no Stage View o ponto de falha e consulte o Console Output.'
        }
        always {
            archiveArtifacts artifacts: 'target/site/jacoco/**,target/pmd.html,frontend/cypress/screenshots/**,frontend/cypress/videos/**', allowEmptyArchive: true
        }
    }
}
