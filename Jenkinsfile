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
    
    stage('Diagnostic Cluster') {
      steps {
        script {
          sh '''
          echo "=== Diagnostic du cluster ==="
          mkdir -p .kube
          cp /var/lib/jenkins/.kube/config .kube/config
          
          echo "1. Vérification des nœuds et de leurs taints:"
          kubectl get nodes -o wide --kubeconfig=.kube/config
          echo ""
          echo "2. Détails des taints:"
          kubectl describe nodes --kubeconfig=.kube/config | grep -A 5 -B 5 Taints || true
          echo ""
          echo "3. Taints exacts:"
          kubectl get nodes -o jsonpath="{.items[*].spec.taints}" --kubeconfig=.kube/config
          echo ""
          
          echo "4. Vérification des pods existants:"
          kubectl get pods -A --kubeconfig=.kube/config | grep -v Completed || true
          '''
        }
      }
    }
    
    stage('Déploiement en dev') {
      steps {
        script {
          sh '''
          echo "=== Déploiement en développement ==="
          
          # Réutiliser le kubeconfig
          KUBECONFIG=".kube/config"
          
          # Créer namespace dev
          kubectl create namespace dev --dry-run=client -o yaml | kubectl apply -f - --kubeconfig=$KUBECONFIG || true
          
          # Nettoyer les anciens déploiements
          echo "Nettoyage des anciens déploiements..."
          kubectl delete deployment app-fastapi -n dev --ignore-not-found=true --kubeconfig=$KUBECONFIG
          kubectl delete replicaset -n dev -l app.kubernetes.io/name=fastapi --ignore-not-found=true --kubeconfig=$KUBECONFIG
          
          # Attendre que la suppression soit effective
          echo "Attente de la suppression des anciens pods..."
          sleep 10
          
          # Créer un fichier values complet AVEC NODE SELECTOR
          cat > values-dev.yml << EOF
replicaCount: 1

image:
  repository: guessod/datascientestapi
  pullPolicy: IfNotPresent
  tag: "${DOCKER_TAG}"

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

podAnnotations: {}
podLabels: {}

podSecurityContext: {}

securityContext: {}

service:
  type: NodePort
  port: 80

ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []

resources: {}

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80

# SOLUTION: Forcer l'exécution sur le master avec nodeSelector et tolerations
nodeSelector:
  node-role.kubernetes.io/control-plane: ""

tolerations:
- key: "node-role.kubernetes.io/control-plane"
  operator: "Exists"
  effect: "NoSchedule"
- key: "node-role.kubernetes.io/master"
  operator: "Exists"
  effect: "NoSchedule"
- key: "node.kubernetes.io/not-ready"
  operator: "Exists"
  effect: "NoSchedule"
- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoSchedule"

affinity: {}
EOF
          
          echo "Valeurs utilisées pour le déploiement:"
          cat values-dev.yml | grep -E "(tag:|nodeSelector:|tolerations:)"
          
          # Déployer avec Helm SANS --wait --timeout
          echo "Exécution de Helm upgrade/install..."
          helm upgrade --install app fastapi \
            --values=values-dev.yml \
            --namespace dev \
            --kubeconfig=$KUBECONFIG \
            --create-namespace
          
          # Attendre manuellement que les pods démarrent
          echo "Attente du démarrage des pods (vérification toutes les 10 secondes)..."
          MAX_ATTEMPTS=30  # 30 * 10 = 300 secondes = 5 minutes max
          for i in $(seq 1 $MAX_ATTEMPTS); do
            POD_STATUS=$(kubectl get pods -n dev -l app.kubernetes.io/name=fastapi -o jsonpath='{.items[*].status.phase}' --kubeconfig=$KUBECONFIG 2>/dev/null || echo "")
            
            if echo "$POD_STATUS" | grep -q "Running"; then
              echo "✓ Pod(s) en cours d'exécution à la tentative $i"
              break
            elif echo "$POD_STATUS" | grep -q "Pending"; then
              echo "⏳ Pod(s) en attente... tentative $i/$MAX_ATTEMPTS"
            elif echo "$POD_STATUS" | grep -q "ContainerCreating"; then
              echo "🔧 Pod(s) en création... tentative $i/$MAX_ATTEMPTS"
            else
              echo "? Statut: $POD_STATUS - tentative $i/$MAX_ATTEMPTS"
            fi
            
            if [ $i -eq $MAX_ATTEMPTS ]; then
              echo "⚠️  Délai d'attente maximal atteint, poursuite du pipeline..."
              break
            fi
            
            sleep 10
          done
          
          echo "Vérification finale du déploiement..."
          kubectl get pods,svc -n dev --kubeconfig=$KUBECONFIG
          
          echo "Détails des pods:"
          kubectl describe pods -n dev --selector=app.kubernetes.io/name=fastapi --kubeconfig=$KUBECONFIG 2>/dev/null | head -100 || true
          
          # Vérifier les logs seulement si le pod est Running
          POD_NAME=$(kubectl get pods -n dev -l app.kubernetes.io/name=fastapi -o jsonpath='{.items[0].metadata.name}' --kubeconfig=$KUBECONFIG 2>/dev/null || echo "")
          if [ -n "$POD_NAME" ]; then
            POD_PHASE=$(kubectl get pod $POD_NAME -n dev -o jsonpath='{.status.phase}' --kubeconfig=$KUBECONFIG 2>/dev/null || echo "")
            if [ "$POD_PHASE" = "Running" ]; then
              echo "Logs du pod $POD_NAME:"
              kubectl logs $POD_NAME -n dev --kubeconfig=$KUBECONFIG --tail=20 || true
            else
              echo "Pod $POD_NAME en phase: $POD_PHASE"
              echo "Événements récents:"
              kubectl get events -n dev --sort-by='.lastTimestamp' --field-selector involvedObject.name=$POD_NAME --kubeconfig=$KUBECONFIG | tail -10 || true
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
          
          # Réutiliser le même kubeconfig
          KUBECONFIG=".kube/config"
          
          # Créer namespace staging
          kubectl create namespace staging --dry-run=client -o yaml | kubectl apply -f - --kubeconfig=$KUBECONFIG || true
          
          # Nettoyer les anciens déploiements
          kubectl delete deployment app-fastapi -n staging --ignore-not-found=true --kubeconfig=$KUBECONFIG
          sleep 5
          
          # Créer values avec nodeSelector
          cat > values-staging.yml << EOF
replicaCount: 2

image:
  repository: guessod/datascientestapi
  pullPolicy: Always
  tag: "${DOCKER_TAG}"

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

podAnnotations: {}
podLabels: {}

podSecurityContext: {}

securityContext: {}

service:
  type: NodePort
  port: 80

ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []

resources: {}

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80

# SOLUTION: Forcer l'exécution sur le master avec nodeSelector et tolerations
nodeSelector:
  node-role.kubernetes.io/control-plane: ""

tolerations:
- key: "node-role.kubernetes.io/control-plane"
  operator: "Exists"
  effect: "NoSchedule"
- key: "node-role.kubernetes.io/master"
  operator: "Exists"
  effect: "NoSchedule"
- key: "node.kubernetes.io/not-ready"
  operator: "Exists"
  effect: "NoSchedule"
- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoSchedule"

affinity: {}
EOF
          
          echo "Déploiement avec Helm..."
          helm upgrade --install app fastapi \
            --values=values-staging.yml \
            --namespace staging \
            --kubeconfig=$KUBECONFIG \
            --create-namespace
          
          # Attente simplifiée
          echo "Attente du démarrage des pods..."
          sleep 30
          
          echo "Vérification staging..."
          kubectl get pods -n staging --kubeconfig=$KUBECONFIG
          
          echo "Statut des pods:"
          kubectl get pods -n staging -l app.kubernetes.io/name=fastapi -o wide --kubeconfig=$KUBECONFIG || true
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
          KUBECONFIG=".kube/config"
          
          # Créer namespace prod
          kubectl create namespace prod --dry-run=client -o yaml | kubectl apply -f - --kubeconfig=$KUBECONFIG || true
          
          # Nettoyer les anciens déploiements
          kubectl delete deployment app-fastapi -n prod --ignore-not-found=true --kubeconfig=$KUBECONFIG
          sleep 5
          
          # Créer un fichier d'overrides complet
          cat > prod-overrides.yml << EOF
image:
  repository: guessod/datascientestapi
  tag: ${DOCKER_TAG}
  pullPolicy: Always
replicaCount: 3
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 200m
    memory: 256Mi

# SOLUTION: Forcer l'exécution sur le master avec nodeSelector et tolerations
nodeSelector:
  node-role.kubernetes.io/control-plane: ""

tolerations:
- key: "node-role.kubernetes.io/control-plane"
  operator: "Exists"
  effect: "NoSchedule"
- key: "node-role.kubernetes.io/master"
  operator: "Exists"
  effect: "NoSchedule"
- key: "node.kubernetes.io/not-ready"
  operator: "Exists"
  effect: "NoSchedule"
- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoSchedule"
EOF
          
          # Déployer avec Helm - utiliser seulement le fichier d'overrides
          helm upgrade --install app fastapi \
            --namespace prod \
            --kubeconfig=$KUBECONFIG \
            --create-namespace \
            --values=prod-overrides.yml
          
          echo "Attente du démarrage des pods..."
          sleep 30
          
          echo "Vérification production..."
          kubectl get pods,svc -n prod --kubeconfig=$KUBECONFIG
          
          echo "État des pods:"
          kubectl get pods -n prod -l app.kubernetes.io/name=fastapi -o wide --kubeconfig=$KUBECONFIG || true
          
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
        echo "Fichiers générés:"
        ls -la *.yml 2>/dev/null || true
        
        echo "État final des déploiements:"
        for ns in dev staging prod; do
          echo "--- Namespace $ns ---"
          kubectl get pods -n $ns --kubeconfig=.kube/config 2>/dev/null || echo "Namespace $ns non accessible"
          RUNNING_PODS=$(kubectl get pods -n $ns -l app.kubernetes.io/name=fastapi --field-selector=status.phase=Running --kubeconfig=.kube/config 2>/dev/null | grep -c Running || echo "0")
          if [ "$RUNNING_PODS" -gt "0" ]; then
            echo "✓ $RUNNING_PODS pod(s) en cours d'exécution dans $ns"
          else
            echo "✗ Aucun pod n'est en cours d'exécution dans $ns"
            echo "Derniers événements:"
            kubectl get events -n $ns --sort-by='.lastTimestamp' --kubeconfig=.kube/config | tail -3 2>/dev/null || true
          fi
        done
        '''
      }
    }
    success {
      script {
        sh '''
        echo "✓ Pipeline exécuté avec succès!"
        echo "Résumé:"
        echo "1. Image Docker construite et push: guessod/datascientestapi:${DOCKER_TAG}"
        echo "2. Déploiements Kubernetes mis à jour dans dev, staging, prod"
        echo "3. Vérifiez l'état des pods avec: kubectl get pods -A"
        '''
      }
    }
    failure {
      script {
        sh '''
        echo "✗ Pipeline en échec"
        echo "Informations de débogage:"
        echo "1. Nœuds disponibles:"
        kubectl get nodes --kubeconfig=.kube/config 2>/dev/null || true
        echo ""
        echo "2. Services dans dev:"
        kubectl get svc -n dev --kubeconfig=.kube/config 2>/dev/null || true
        '''
      }
    }
  }
}
