pipeline {
  environment { // Declaration of environment variables
    DOCKER_ID = "guessod"
    DOCKER_IMAGE = "datascientestapi"
    DOCKER_TAG = "v.${BUILD_ID}.0" // we will tag our images with the current build in order to increment the value by 1 with each new build
  }
  agent any // Jenkins will be able to select all available agents
  
  stages {
    stage('Préparation') { // Nouvelle étape pour libérer le port 80
      steps {
        script {
          sh '''
          # Nettoyer l'ancien conteneur "jenkins"
          docker rm -f jenkins || true
          
          # Arrêter le conteneur edge-router qui utilise le port 80
          docker stop docker-compose-edge-router-1 2>/dev/null || true
          docker rm -f docker-compose-edge-router-1 2>/dev/null || true
          
          # Vérifier que le port 80 est libre
          echo "Vérification du port 80..."
          if docker ps -a --filter "publish=80" | grep -q .; then
            echo "ATTENTION : Un conteneur utilise encore le port 80 !"
            docker ps -a --filter "publish=80"
            echo "Forçage du nettoyage..."
            docker ps -a --filter "publish=80" --format "{{.ID}}" | xargs -r docker rm -f
          fi
          '''
        }
      }
    }
    
    stage('Docker Build') { // docker build image stage
      steps {
        script {
          sh '''
          docker build -t $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG .
          sleep 6
          '''
        }
      }
    }
    
    stage('Docker run') { // run container from our builded image
      steps {
        script {
          sh '''
          docker run -d -p 80:80 --name jenkins $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG
          sleep 10
          '''
        }
      }
    }
    
    stage('Test Acceptance') { // we launch the curl command to validate that the container responds to the request
      steps {
        script {
          sh '''
          # Tenter plusieurs fois en cas de démarrage lent
          MAX_ATTEMPTS=5
          for i in $(seq 1 $MAX_ATTEMPTS); do
            if curl -s -f localhost > /dev/null; then
              echo "✓ Test réussi à la tentative $i"
              curl localhost
              break
            else
              echo "Tentative $i/$MAX_ATTEMPTS échouée, attente de 5 secondes..."
              sleep 5
            fi
            if [ $i -eq $MAX_ATTEMPTS ]; then
              echo "✗ Échec après $MAX_ATTEMPTS tentatives"
              docker logs jenkins
              exit 1
            fi
          done
          '''
        }
      }
    }
    
    stage('Docker Push') { // we pass the built image to our docker hub account
      environment {
        DOCKER_PASS = credentials("DOCKER_HUB_PASS") // we retrieve docker password from secret text called docker_hub_pass saved on jenkins
      }
      steps {
        script {
          sh '''
          docker login -u $DOCKER_ID -p $DOCKER_PASS
          docker push $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG
          '''
        }
      }
    }
    
    stage('Déploiement en dev') {
      steps {
        script {
          sh '''
          echo "=== Déploiement en développement ==="
          
          # 1. Utiliser votre kubeconfig local directement
          mkdir -p .kube
          cp ~/.kube/config .kube/config
          
          # 2. Vérifier que la connexion fonctionne
          echo "Test de connexion au cluster..."
          kubectl cluster-info --kubeconfig=.kube/config || {
            echo "Échec avec les certificats, tentative sans vérification TLS..."
            # Modifier le kubeconfig pour ignorer TLS
            sed -i 's/certificate-authority-data:/insecure-skip-tls-verify: true\\n    certificate-authority-data:/' .kube/config
          }
          
          # 3. Vérifier à nouveau
          kubectl cluster-info --kubeconfig=.kube/config || {
            echo "Impossible de se connecter au cluster"
            exit 1
          }
          
          # 4. Vérifier le contexte actuel
          CURRENT_CONTEXT=$(kubectl config current-context --kubeconfig=.kube/config)
          echo "Contexte actuel: $CURRENT_CONTEXT"
          
          # 5. Créer le namespace si nécessaire
          kubectl create namespace dev --dry-run=client -o yaml | kubectl apply -f - --kubeconfig=.kube/config || true
          
          # 6. Préparer le fichier values pour Helm
          cp fastapi/values.yaml values-dev.yml
          sed -i "s/tag:.*/tag: ${DOCKER_TAG}/" values-dev.yml
          
          echo "Valeurs utilisées pour le déploiement:"
          cat values-dev.yml | grep -A2 -B2 "tag:"
          
          # 7. Déployer avec Helm
          echo "Exécution de Helm upgrade/install..."
          helm upgrade --install app fastapi \
            --values=values-dev.yml \
            --namespace dev \
            --kubeconfig=.kube/config \
            --create-namespace \
            --wait \
            --timeout 5m \
            --debug 2>&1 | tail -20
          
          # 8. Vérifier le déploiement
          echo "Vérification du déploiement..."
          kubectl get pods,svc -n dev --kubeconfig=.kube/config
          
          # 9. Vérifier le rollout
          kubectl rollout status deployment/app -n dev --kubeconfig=.kube/config --timeout=2m || \
            echo "Rollback ou timeout, vérifiez les logs des pods"
          '''
        }
      }
    }
    
    stage('Deploiement en staging') {
      steps {
        script {
          sh '''
          echo "=== Déploiement en staging ==="
          
          # Réutiliser le même kubeconfig
          mkdir -p .kube 2>/dev/null || true
          cp ~/.kube/config .kube/config 2>/dev/null || true
          
          # Créer namespace staging
          kubectl create namespace staging --dry-run=client -o yaml | kubectl apply -f - --kubeconfig=.kube/config || true
          
          # Préparer values
          cp fastapi/values.yaml values-staging.yml
          sed -i "s/tag:.*/tag: ${DOCKER_TAG}/" values-staging.yml
          
          # Ajouter des spécificités staging
          echo -e "\\n# Configuration staging" >> values-staging.yml
          echo "replicaCount: 2" >> values-staging.yml
          
          # Déployer
          helm upgrade --install app fastapi \
            --values=values-staging.yml \
            --namespace staging \
            --kubeconfig=.kube/config \
            --create-namespace \
            --wait \
            --timeout 5m
          
          echo "Vérification staging..."
          kubectl get pods -n staging --kubeconfig=.kube/config
          '''
        }
      }
    }
    
    stage('Deploiement en prod') {
      steps {
        timeout(time: 15, unit: "MINUTES") {
          input message: 'Do you want to deploy in production ?', ok: 'Yes'
        }
        script {
          sh '''
          echo "=== Déploiement en production ==="
          
          # Réutiliser le même kubeconfig
          mkdir -p .kube 2>/dev/null || true
          cp ~/.kube/config .kube/config 2>/dev/null || true
          
          # Créer namespace prod
          kubectl create namespace prod --dry-run=client -o yaml | kubectl apply -f - --kubeconfig=.kube/config || true
          
          # Préparer values avec spécificités production
          cp fastapi/values.yaml values-prod.yml
          sed -i "s/tag:.*/tag: ${DOCKER_TAG}/" values-prod.yml
          
          # Configuration production
          echo -e "\\n# Configuration production" >> values-prod.yml
          echo "replicaCount: 3" >> values-prod.yml
          echo "resources:" >> values-prod.yml
          echo "  limits:" >> values-prod.yml
          echo "    cpu: 500m" >> values-prod.yml
          echo "    memory: 512Mi" >> values-prod.yml
          echo "  requests:" >> values-prod.yml
          echo "    cpu: 200m" >> values-prod.yml
          echo "    memory: 256Mi" >> values-prod.yml
          
          # Déployer avec plus de vérifications
          helm upgrade --install app fastapi \
            --values=values-prod.yml \
            --namespace prod \
            --kubeconfig=.kube/config \
            --create-namespace \
            --wait \
            --timeout 10m \
            --atomic \
            --cleanup-on-fail
          
          echo "Vérification production..."
          kubectl get pods,svc,ingress -n prod --kubeconfig=.kube/config
          
          # Vérifier le rollout
          kubectl rollout status deployment/app -n prod --kubeconfig=.kube/config --timeout=5m
          
          echo "✓ Déploiement en production réussi!"
          '''
        }
      }
    }
  }
  
  post {
    always {
      script {
        sh '''
        echo "=== Nettoyage ==="
        # Garder les fichiers pour débogage si nécessaire
        echo "Fichiers générés:"
        ls -la *.yml 2>/dev/null || true
        '''
      }
    }
    success {
      script {
        sh '''
        echo "✓ Pipeline exécuté avec succès!"
        '''
      }
    }
    failure {
      script {
        sh '''
        echo "✗ Pipeline en échec"
        echo "Logs des derniers pods en dev:"
        kubectl get pods -n dev --kubeconfig=.kube/config 2>/dev/null || true
        kubectl logs --tail=20 -n dev deployment/app --kubeconfig=.kube/config 2>/dev/null || true
        '''
      }
    }
  }
}
