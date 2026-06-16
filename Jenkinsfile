// Jenkinsfile -- Pipeline CI/CD SentimentAI
pipeline {
  agent any  // s'exécute sur n'importe quel agent disponible

  environment {
    IMAGE_NAME = 'sentiment-ai'
    REGISTRY   = 'ghcr.io/VOTRE_PSEUDO'  // remplacez VOTRE_PSEUDO
    // IMAGE_TAG = SHA Git court du commit (ex: a3f8c12)
    // Chaque build produit une image taguée de façon unique et traçable
    IMAGE_TAG  = sh(script: 'git rev-parse --short HEAD',
                    returnStdout: true).trim()
  }

  stages {

    stage('Checkout') {
      steps {
        // Jenkins clone automatiquement le repo configuré dans le job
        checkout scm
        // Afficher des infos de traçabilité dans les logs
        echo "Branche : ${env.BRANCH_NAME}"
        echo "Commit  : ${env.GIT_COMMIT}"
        sh 'git log --oneline -5'
      }
    }

    stage('Lint') {
      steps {
        // Lancer flake8 dans un conteneur Python temporaire
        // --rm supprime le conteneur automatiquement après l'exécution
        sh ''
          docker run --rm \
            --volumes-from jenkins \
            -v $WORKSPACE \
            python:3.12-slim \
            sh -c "pip install flake8 -q && flake8 src/ --max-line-length=100"
        '''
      }
    }

    stage('Build & Test') {
      steps {
        // Construire l'image Docker taguée avec le SHA Git
        sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."

        // Lancer pytest dans le conteneur fraîchement construit
        sh """
          docker run --rm \
            ${IMAGE_NAME}:${IMAGE_TAG} \
            pytest tests/ -v \
              --cov=src \
              --cov-report=xml:coverage.xml \
              --cov-report=term-missing \
              --cov-fail-under=70
        """
      }
      post {
        failure {
          echo 'Tests échoués ou coverage insuffisant (< 70%)'
        }
      }
    }

    stage('Push') {
      // Ce stage ne s'exécute QUE sur la branche main
      when { branch 'main' }
      steps {
        // Se connecter au registry avec les credentials Jenkins (jamais en clair !)
        withCredentials([usernamePassword(
            credentialsId: 'github-token',
            usernameVariable: 'REGISTRY_USER',
            passwordVariable: 'REGISTRY_PASS'
        )]) {
          sh '''
            # Connexion au registry GitHub Container Registry
            echo $REGISTRY_PASS | docker login ghcr.io \
              -u $REGISTRY_USER --password-stdin

            # Pousser l'image taguée avec le SHA Git (immuable, traçable)
            docker push ''' + "${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}" + '''

            # Pousser aussi le tag :latest pour faciliter les pulls manuels
            docker tag ''' + "${IMAGE_NAME}:${IMAGE_TAG}" + \
              " ${REGISTRY}/${IMAGE_NAME}:latest" + '''
            docker push ''' + "${REGISTRY}/${IMAGE_NAME}:latest" + '''
          '''
        }
      }
    }

  }

  post {
    always {
      // Nettoyer les conteneurs de test, qu'il y ait succès ou échec
      sh 'docker compose down -v 2>/dev/null || true'
    }
    success {
      echo "Pipeline réussi ! Image : ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
    }
    failure {
      echo 'Pipeline échoué. Consultez les logs ci-dessus.'
    }
  }
}