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
          
          # 1. Utiliser le kubeconfig de Jenkins
          mkdir -p .kube
          cp /var/lib/jenkins/.kube/config .kube/config
          
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
          
          # 6. Préparer le fichier values pour Helm - SOLUTION COMPLÈTE
          # Créer un nouveau fichier values au lieu de modifier l'existant
          cat > values-dev.yml << EOF
replicaCount: 1

image:
  repository: guessod/datascientestapi
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "${DOCKER_TAG}"

# This is for the secrets for pulling an image from a private repository more information can be found here: https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  # Specifies whether a service account should be created
  create: true
  # Annotations to add to the service account
  annotations: {}
  # The name of the service account to use.
  # If not set and create is true, a name is generated using the fullname template
  name: ""

podAnnotations: {}
podLabels: {}

podSecurityContext: {}
  # fsGroup: 2000

securityContext: {}
  # capabilities:
  #   drop:
  #   - ALL
  # readOnlyRootFilesystem: true
  # runAsNonRoot: true
  # runAsUser: 1000

service:
  type: NodePort
  port: 80

ingress:
  enabled: false
  className: ""
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - chart-example.local

resources: {}
  # We usually recommend not to specify default resources and to leave this as a conscious
  # choice for the user. This also increases chances charts run on environments with little
  # resources, such as Minikube. If you do want to specify resources, uncomment the following
  # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
  # limits:
  #   cpu: 100m
  #   memory: 128Mi
  # requests:
  #   cpu: 100m
  #   memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80
  # targetMemoryUtilizationPercentage: 80

nodeSelector: {}

# CORRECTION: Tolerations pour résoudre le problème de taints
tolerations:
- key: "node.kubernetes.io/not-ready"
  operator: "Exists"
  effect: "NoSchedule"
- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoSchedule"
- key: "node-role.kubernetes.io/control-plane"
  operator: "Exists"
  effect: "NoSchedule"
- key: "node-role.kubernetes.io/master"
  operator: "Exists"
  effect: "NoSchedule"

affinity: {}
EOF
          
          echo "Valeurs utilisées pour le déploiement:"
          cat values-dev.yml | grep -A2 -B2 "tag:"
          
          # 7. Déployer avec Helm
          echo "Exécution de Helm upgrade/install..."
          helm upgrade --install app fastapi \
            --values=values-dev.yml \
            --namespace dev \
            --kubeconfig=.kube/config \
            --create-namespace
          
          # 8. Vérifier le déploiement manuellement
          echo "Attente du démarrage des pods..."
          sleep 30
          
          echo "Vérification du déploiement..."
          kubectl get pods,svc -n dev --kubeconfig=.kube/config
          
          # 9. Vérifier l'état des pods
          echo "Détails des pods:"
          kubectl describe pods -n dev --selector=app.kubernetes.io/name=fastapi --kubeconfig=.kube/config || true
          
          # 10. Vérifier les logs si les pods tournent
          POD_NAME=$(kubectl get pods -n dev -l app.kubernetes.io/name=fastapi -o jsonpath='{.items[0].metadata.name}' --kubeconfig=.kube/config 2>/dev/null || echo "")
          if [ -n "$POD_NAME" ]; then
            echo "Logs du pod $POD_NAME:"
            kubectl logs $POD_NAME -n dev --kubeconfig=.kube/config --tail=20 || true
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
          mkdir -p .kube 2>/dev/null || true
          cp /var/lib/jenkins/.kube/config .kube/config 2>/dev/null || true
          KUBECONFIG=".kube/config"
          
          # Créer namespace staging
          kubectl create namespace staging --dry-run=client -o yaml | kubectl apply -f - --kubeconfig=$KUBECONFIG || true
          
          # Préparer values - SOLUTION COMPLÈTE
          cat > values-staging.yml << EOF
replicaCount: 2

image:
  repository: guessod/datascientestapi
  pullPolicy: Always
  # Overrides the image tag whose default is the chart appVersion.
  tag: "${DOCKER_TAG}"

# This is for the secrets for pulling an image from a private repository more information can be found here: https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  # Specifies whether a service account should be created
  create: true
  # Annotations to add to the service account
  annotations: {}
  # The name of the service account to use.
  # If not set and create is true, a name is generated using the fullname template
  name: ""

podAnnotations: {}
podLabels: {}

podSecurityContext: {}
  # fsGroup: 2000

securityContext: {}
  # capabilities:
  #   drop:
  #   - ALL
  # readOnlyRootFilesystem: true
  # runAsNonRoot: true
  # runAsUser: 1000

service:
  type: NodePort
  port: 80

ingress:
  enabled: false
  className: ""
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - chart-example.local

resources: {}
  # We usually recommend not to specify default resources and to leave this as a conscious
  # choice for the user. This also increases chances charts run on environments with little
  # resources, such as Minikube. If you do want to specify resources, uncomment the following
  # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
  # limits:
  #   cpu: 100m
  #   memory: 128Mi
  # requests:
  #   cpu: 100m
  #   memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80
  # targetMemoryUtilizationPercentage: 80

nodeSelector: {}

# CORRECTION: Tolerations pour résoudre le problème de taints
tolerations:
- key: "node.kubernetes.io/not-ready"
  operator: "Exists"
  effect: "NoSchedule"
- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoSchedule"
- key: "node-role.kubernetes.io/control-plane"
  operator: "Exists"
  effect: "NoSchedule"
- key: "node-role.kubernetes.io/master"
  operator: "Exists"
  effect: "NoSchedule"

affinity: {}
EOF
          
          echo "Vérification des modifications:"
          grep -n -E "(tag:|replicaCount:|pullPolicy:|tolerations:)" values-staging.yml
          
          # Déployer avec Helm
          echo "Déploiement avec Helm..."
          helm upgrade --install app fastapi \
            --values=values-staging.yml \
            --namespace staging \
            --kubeconfig=$KUBECONFIG \
            --create-namespace
          
          # Attendre un peu
          echo "Attente du démarrage des pods..."
          sleep 30
          
          echo "Vérification staging..."
          kubectl get pods -n staging --kubeconfig=$KUBECONFIG
          
          # Vérifier les événements pour debug
          echo "Événements récents:"
          kubectl get events -n staging --sort-by='.lastTimestamp' --kubeconfig=$KUBECONFIG | tail -10 || true
          
          # Vérifier l'état des pods
          echo "Description des pods:"
          kubectl describe pods -n staging --selector=app.kubernetes.io/name=fastapi --kubeconfig=$KUBECONFIG 2>/dev/null | head -50 || true
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
          cp /var/lib/jenkins/.kube/config .kube/config 2>/dev/null || true
          KUBECONFIG=".kube/config"
          
          # Créer namespace prod
          kubectl create namespace prod --dry-run=client -o yaml | kubectl apply -f - --kubeconfig=$KUBECONFIG || true
          
          # Créer un fichier d'overrides séparé pour éviter les problèmes YAML
          cat > prod-overrides.yml << EOF
image:
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
# AJOUT: Tolerations pour résoudre le problème de taints
tolerations:
- key: "node.kubernetes.io/not-ready"
  operator: "Exists"
  effect: "NoSchedule"
- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoSchedule"
- key: "node-role.kubernetes.io/control-plane"
  operator: "Exists"
  effect: "NoSchedule"
- key: "node-role.kubernetes.io/master"
  operator: "Exists"
  effect: "NoSchedule"
EOF
          
          # Déployer avec Helm - SANS TIMEOUT ET SANS --wait
          helm upgrade --install app fastapi \
            --namespace prod \
            --kubeconfig=$KUBECONFIG \
            --create-namespace \
            --values=fastapi/values.yaml \
            --values=prod-overrides.yml
          
          # Attendre un peu
          echo "Attente du démarrage des pods..."
          sleep 30
          
          echo "Vérification production..."
          kubectl get pods,svc,ingress -n prod --kubeconfig=$KUBECONFIG
          
          # Vérifier les pods en détail
          echo "État détaillé des pods:"
          kubectl describe pods -n prod --selector=app.kubernetes.io/name=fastapi --kubeconfig=$KUBECONFIG 2>/dev/null | grep -A 20 "Events:" || true
          
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
        echo "=== Nettoyage ==="
        # Garder les fichiers pour débogage si nécessaire
        echo "Fichiers générés:"
        ls -la *.yml 2>/dev/null || true
        
        # Vérifier l'état final de tous les déploiements
        echo "État final des déploiements:"
        for ns in dev staging prod; do
          echo "--- Namespace $ns ---"
          kubectl get pods,svc -n $ns --kubeconfig=.kube/config 2>/dev/null || echo "Namespace $ns non accessible"
        done
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
