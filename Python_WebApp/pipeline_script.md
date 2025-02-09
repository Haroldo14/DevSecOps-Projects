pipeline{
    agent any  <!-- Exécute le pipeline sur n'importe quel agent Jenkins disponible -->
    tools{
        jdk 'jdk17' <!-- Spécifie que le pipeline utilise Java 17 -->
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'  <!-- Définit la variable d'environnement pour SonarQube Scanner -->
    }
    stages {
        stage('clean workspace'){  <!-- Nettoyage de l'espace de travail pour éviter les fichiers obsolètes -->
            steps{
                cleanWs()
            }
        }
        stage('Checkout From Git'){  <!-- Récupération du code source depuis GitHub -->
            steps{
                git branch: 'main', url: 'https://github.com/Aj7Ay/Python-System-Monitoring.git'
            }
        }
        stage("Sonarqube Analysis "){  <!-- Analyse du code source avec SonarQube -->
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Python-Webapp \
                    -Dsonar.projectKey=Python-Webapp '''
                }
            }
        }
        stage("quality gate"){  <!-- Vérification que le code respecte les standards de qualité -->
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage("TRIVY File scan"){ <!-- Scan de sécurité des fichiers du projet avec Trivy -->
            steps{
                sh "trivy fs . > trivy-fs_report.txt"
            }
        }
        stage("OWASP Dependency Check"){ <!-- Scan des dépendances avec OWASP Dependency Check -->
            steps{
                dependencyCheck additionalArguments: '--scan ./ --format XML ', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        <!-- Fin premiere partie Marqueur 1--> 
        <!-- Debut deuxime partie mArquer 2-->
        stage("Docker Build & tag"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       sh "make image" <!-- Exécute la commande `make image` pour construire l’image -->
                    }
                }
            }
        }
        stage("TRIVY"){ <!-- Analyse de l’image Docker avec Trivy -->
            steps{
                sh "trivy image sevenajay/python-system-monitoring:latest > trivy.txt"
            }
        }
        stage("Docker Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       //sh "make push"
                       sh "docker tag sevenajay/python-system-monitoring:latest haroldo1414/python-system-monitoring:latest" <!-- Renomer image-->
                       sh "docker push haroldo1414/python-system-monitoring:latest" <!-- Push de l'image vers mon dockerhub -->
                    }
                }
            }
        }
        stage("Deploy to container"){
            steps{
                sh "docker run -d --name python1 -p 5000:5000 haroldo1414/python-system-monitoring:latest" <!-- Deploiement de l'application sur le port 5000 utilisant l'image via notre dockerhub -->
            }
        }
    }
}