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
    
    stage('Diagnostic et Nettoyage Cluster') {
      steps {
        script {
          sh '''
          echo "=== Diagnostic et nettoyage du cluster ==="
          mkdir -p .kube
          cp /var/lib/jenkins/.kube/config .kube/config
          KUBECONFIG=".kube/config"
          
          # 1. Vérifier l'état du nœud
          echo "1. État du nœud:"
          kubectl get nodes --kubeconfig=$KUBECONFIG
          echo ""
          
          # 2. Vérifier les taints
          echo "2. Taints actuels:"
          kubectl describe nodes --kubeconfig=$KUBECONFIG | grep -i taint || echo "Aucun taint"
          echo ""
          
          # 3. SOLUTION: Supprimer le taint de disk-pressure
          echo "3. Suppression du taint 'disk-pressure'..."
          kubectl taint nodes --all node.kubernetes.io/disk-pressure:NoSchedule- --kubeconfig=$KUBECONFIG
          
          # 4. Vérifier l'espace disque
          echo "4. Vérification de l'espace disque du nœud..."
          kubectl describe nodes --kubeconfig=$KUBECONFIG | grep -A 5 -B 5 "Allocated resources:" || true
          echo ""
          
          # 5. Nettoyer les anciens pods terminés
          echo "5. Nettoyage des pods terminés..."
          for ns in dev staging prod kube-system; do
            echo "--- Namespace $ns ---"
            # Supprimer les pods en erreur/terminés
            kubectl delete pods -n $ns --field-selector=status.phase!=Running --kubeconfig=$KUBECONFIG 2>/dev/null || true
          done
          
          # 6. Supprimer les anciens déploiements
          echo "6. Suppression des anciens déploiements..."
          for ns in dev staging prod; do
            kubectl delete deployment app-fastapi -n $ns --ignore-not-found=true --kubeconfig=$KUBECONFIG 2>/dev/null || true
            sleep 2
          done
          
          # 7. Vérification finale
          echo "7. État final des taints:"
          kubectl describe nodes --kubeconfig=$KUBECONFIG | grep -i taint || echo "✓ Aucun taint restant"
          '''
        }
      }
    }
    
    stage('Déploiement en dev') {
      steps {
        script {
          sh '''
          echo "=== Déploiement en développement ==="
          KUBECONFIG=".kube/config"
          
          # Créer un fichier values SIMPLIFIÉ avec toleration pour disk-pressure
          cat > values-dev.yml << EOF
replicaCount: 1

image:
  repository: guessod/datascientestapi
  pullPolicy: IfNotPresent
  tag: "${DOCKER_TAG}"

service:
  type: NodePort
  port: 80

# Toleration pour disk-pressure
tolerations:
- key: "node.kubernetes.io/disk-pressure"
  operator: "Exists"
  effect: "NoSchedule"

resources:
  requests:
    memory: "64Mi"
    cpu: "50m"
  limits:
    memory: "128Mi"
    cpu: "100m"
EOF
          
          echo "Valeurs utilisées:"
          cat values-dev.yml | grep -E "(tag:|tolerations:|memory:|cpu:)"
          
          # Déployer avec Helm
          echo "Exécution de Helm upgrade/install..."
          helm upgrade --install app fastapi \
            --values=values-dev.yml \
            --namespace dev \
            --kubeconfig=$KUBECONFIG \
            --create-namespace
          
          # Attendre et vérifier
          echo "Attente du démarrage du pod..."
          for i in $(seq 1 30); do  # 30 * 2 = 60 secondes max
            POD_STATUS=$(kubectl get pods -n dev -l app.kubernetes.io/name=fastapi -o jsonpath='{.items[0].status.phase}' --kubeconfig=$KUBECONFIG 2>/dev/null || echo "NotFound")
            if [ "$POD_STATUS" = "Running" ]; then
              echo "✓ Pod en cours d'exécution après ${i}2 secondes"
              break
            elif [ "$POD_STATUS" = "Pending" ]; then
              echo "⏳ Pod en attente... ($i/30)"
            elif [ "$POD_STATUS" = "ContainerCreating" ]; then
              echo "🔧 Pod en création... ($i/30)"
            else
              echo "? Statut: $POD_STATUS ($i/30)"
            fi
            sleep 2
          done
          
          echo "Vérification finale:"
          kubectl get pods,svc -n dev --kubeconfig=$KUBECONFIG
          
          # Afficher les logs si le pod est Running
          POD_NAME=$(kubectl get pods -n dev -l app.kubernetes.io/name=fastapi -o jsonpath='{.items[0].metadata.name}' --kubeconfig=$KUBECONFIG 2>/dev/null || echo "")
          if [ -n "$POD_NAME" ]; then
            echo "Détails du pod:"
            kubectl describe pod $POD_NAME -n dev --kubeconfig=$KUBECONFIG | tail -20
            if kubectl get pod $POD_NAME -n dev --kubeconfig=$KUBECONFIG 2>/dev/null | grep -q Running; then
              echo "Logs du pod:"
              kubectl logs $POD_NAME -n dev --kubeconfig=$KUBECONFIG --tail=10
            fi
          fi
          '''
        }
      }
    }
    
    stage('Deploiement en staging') {
      steps {
        script {
          sh '''
          echo "=== Déploiement en staging ==="
          KUBECONFIG=".kube/config"
          
          cat > values-staging.yml << EOF
replicaCount: 2

image:
  repository: guessod/datascientestapi
  pullPolicy: Always
  tag: "${DOCKER_TAG}"

service:
  type: NodePort
  port: 80

# Toleration pour disk-pressure
tolerations:
- key: "node.kubernetes.io/disk-pressure"
  operator: "Exists"
  effect: "NoSchedule"

resources:
  requests:
    memory: "64Mi"
    cpu: "50m"
  limits:
    memory: "128Mi"
    cpu: "100m"
EOF
          
          echo "Déploiement avec Helm..."
          helm upgrade --install app fastapi \
            --values=values-staging.yml \
            --namespace staging \
            --kubeconfig=$KUBECONFIG \
            --create-namespace
          
          sleep 15
          
          echo "Vérification:"
          kubectl get pods -n staging --kubeconfig=$KUBECONFIG
          
          # Vérifier l'état des pods
          RUNNING_COUNT=$(kubectl get pods -n staging -l app.kubernetes.io/name=fastapi --field-selector=status.phase=Running --kubeconfig=$KUBECONFIG 2>/dev/null | grep -c Running || echo "0")
          echo "Pods en cours d'exécution: $RUNNING_COUNT/2"
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
          KUBECONFIG=".kube/config"
          
          cat > prod-overrides.yml << EOF
image:
  repository: guessod/datascientestapi
  tag: ${DOCKER_TAG}
  pullPolicy: Always
replicaCount: 3

service:
  type: NodePort
  port: 80

# Toleration pour disk-pressure
tolerations:
- key: "node.kubernetes.io/disk-pressure"
  operator: "Exists"
  effect: "NoSchedule"

resources:
  requests:
    memory: "64Mi"
    cpu: "50m"
  limits:
    memory: "128Mi"
    cpu: "100m"
EOF
          
          helm upgrade --install app fastapi \
            --namespace prod \
            --kubeconfig=$KUBECONFIG \
            --create-namespace \
            --values=prod-overrides.yml
          
          sleep 20
          
          echo "Vérification:"
          kubectl get pods -n prod --kubeconfig=$KUBECONFIG
          
          echo "✓ Déploiement en production lancé!"
          '''
        }
      }
    }
  }
  
  post {
    always {
      script {
        sh '''
        echo "=== Rapport final ==="
        KUBECONFIG=".kube/config"
        
        echo "1. État des déploiements:"
        for ns in dev staging prod; do
          echo "--- $ns ---"
          kubectl get pods -n $ns --kubeconfig=$KUBECONFIG 2>/dev/null || echo "Namespace inaccessible"
          TOTAL_PODS=$(kubectl get pods -n $ns -l app.kubernetes.io/name=fastapi --kubeconfig=$KUBECONFIG 2>/dev/null | grep -c -v NAME || echo "0")
          RUNNING_PODS=$(kubectl get pods -n $ns -l app.kubernetes.io/name=fastapi --field-selector=status.phase=Running --kubeconfig=$KUBECONFIG 2>/dev/null | grep -c Running || echo "0")
          echo "Pods: $RUNNING_PODS/$TOTAL_PODS en cours d'exécution"
          echo ""
        done
        
        echo "2. Services exposés:"
        for ns in dev staging prod; do
          NODE_PORT=$(kubectl get svc -n $ns app-fastapi -o jsonpath='{.spec.ports[0].nodePort}' --kubeconfig=$KUBECONFIG 2>/dev/null || echo "N/A")
          echo "$ns: NodePort = $NODE_PORT"
        done
        
        echo "3. État du nœud:"
        kubectl describe nodes --kubeconfig=$KUBECONFIG | grep -E "(Taints:|Allocated resources:)" || true
        '''
      }
    }
    success {
      script {
        sh '''
        echo "✓ Pipeline exécuté avec succès!"
        echo ""
        echo "RÉSUMÉ:"
        echo "1. Image Docker: guessod/datascientestapi:${DOCKER_TAG}"
        echo "2. Taint 'disk-pressure' supprimé du nœud"
        echo "3. Déploiements mis à jour avec tolerations pour disk-pressure"
        echo "4. Accès aux applications:"
        echo "   - dev: curl http://<node-ip>:<node-port-dev>"
        echo "   - staging: curl http://<node-ip>:<node-port-staging>"
        echo "   - prod: curl http://<node-ip>:<node-port-prod>"
        '''
      }
    }
    failure {
      script {
        sh '''
        echo "✗ Pipeline en échec"
        echo ""
        echo "DÉBOGAGE:"
        KUBECONFIG=".kube/config"
        echo "1. Événements récents:"
        kubectl get events -A --sort-by='.lastTimestamp' --kubeconfig=$KUBECONFIG | tail -10 2>/dev/null || true
        echo ""
        echo "2. État des pods:"
        kubectl get pods -A --kubeconfig=$KUBECONFIG | grep -v Completed 2>/dev/null || true
        echo ""
        echo "3. Espace disque:"
        df -h /var/lib/docker /var/lib/kubelet 2>/dev/null || true
        '''
      }
    }
  }
}
