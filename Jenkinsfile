pipeline {
  environment {
    DOCKER_ID = "guessod"
    DOCKER_IMAGE = "datascientestapi"
    DOCKER_TAG = "v.${BUILD_ID}.0"
  }
  agent any
  
  stages {
    stage('Préparation et Nettoyage') {
      steps {
        script {
          sh '''
          echo "=== LIBÉRATION D'ESPACE DISQUE ==="
          
          # Arrêter et nettoyer les conteneurs Docker
          docker rm -f jenkins || true
          docker stop docker-compose-edge-router-1 2>/dev/null || true
          docker rm -f docker-compose-edge-router-1 2>/dev/null || true
          
          # Libérer de l'espace disque
          echo "Nettoyage Docker agressif..."
          docker system prune -a -f
          
          # Vérifier l'espace disque
          echo "Espace disque actuel:"
          df -h /
          '''
        }
      }
    }
    
    stage('Docker Build') {
      steps {
        script {
          sh '''
          docker build -t $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG .
          sleep 6
          '''
        }
      }
    }
    
    stage('Docker run') {
      steps {
        script {
          sh '''
          docker run -d -p 80:80 --name jenkins $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG
          sleep 10
          '''
        }
      }
    }
    
    stage('Test Acceptance') {
      steps {
        script {
          sh '''
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
    
    stage('Docker Push') {
      environment {
        DOCKER_PASS = credentials("DOCKER_HUB_PASS")
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
    
    stage('Gestion du Cluster Kubernetes') {
      steps {
        script {
          sh '''
          echo "=== CONFIGURATION DU CLUSTER KUBERNETES ==="
          mkdir -p .kube
          cp /var/lib/jenkins/.kube/config .kube/config
          KUBECONFIG=".kube/config"
          
          # 1. Supprimer le taint disk-pressure
          echo "1. Suppression du taint disk-pressure..."
          kubectl taint nodes --all node.kubernetes.io/disk-pressure:NoSchedule- --kubeconfig=$KUBECONFIG 2>/dev/null || true
          
          # 2. NETTOYAGE COMPLET - Supprimer tous les déploiements et services existants
          echo "2. Nettoyage complet des ressources existantes..."
          for ns in dev staging prod; do
            echo "--- Nettoyage namespace $ns ---"
            # Supprimer les déploiements
            kubectl delete deployment fastapi-dev -n $ns --ignore-not-found=true --kubeconfig=$KUBECONFIG
            kubectl delete deployment fastapi-staging -n $ns --ignore-not-found=true --kubeconfig=$KUBECONFIG
            kubectl delete deployment fastapi-prod -n $ns --ignore-not-found=true --kubeconfig=$KUBECONFIG
            kubectl delete deployment app-fastapi -n $ns --ignore-not-found=true --kubeconfig=$KUBECONFIG
            
            # Supprimer les services
            kubectl delete svc fastapi-dev -n $ns --ignore-not-found=true --kubeconfig=$KUBECONFIG
            kubectl delete svc fastapi-staging -n $ns --ignore-not-found=true --kubeconfig=$KUBECONFIG
            kubectl delete svc fastapi-prod -n $ns --ignore-not-found=true --kubeconfig=$KUBECONFIG
            kubectl delete svc app-fastapi -n $ns --ignore-not-found=true --kubeconfig=$KUBECONFIG
            
            # Supprimer tous les pods restants
            kubectl delete pods --all -n $ns --kubeconfig=$KUBECONFIG --ignore-not-found=true
          done
          
          # Attendre que tout soit supprimé
          sleep 5
          
          # 3. Créer les namespaces s'ils n'existent pas
          echo "3. Création des namespaces..."
          for ns in dev staging prod; do
            kubectl create namespace $ns --dry-run=client -o yaml --kubeconfig=$KUBECONFIG | kubectl apply -f - --kubeconfig=$KUBECONFIG
          done
          
          # 4. Créer les déploiements AVEC des ports NodePort AUTO (pas fixés)
          echo "4. Création des déploiements avec ports automatiques..."
          
          cat > all-deployments.yaml << EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-dev
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fastapi
      env: dev
  template:
    metadata:
      labels:
        app: fastapi
        env: dev
    spec:
      tolerations:
      - key: "node.kubernetes.io/disk-pressure"
        operator: "Exists"
        effect: "NoSchedule"
      - key: "node.kubernetes.io/memory-pressure"
        operator: "Exists"
        effect: "NoSchedule"
      containers:
      - name: fastapi
        image: guessod/datascientestapi:${DOCKER_TAG}
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "32Mi"
            cpu: "10m"
          limits:
            memory: "64Mi"
            cpu: "50m"
---
apiVersion: v1
kind: Service
metadata:
  name: fastapi-dev
  namespace: dev
spec:
  type: NodePort
  selector:
    app: fastapi
    env: dev
  ports:
  - port: 80
    targetPort: 80
    # NE PAS spécifier nodePort, laisser Kubernetes choisir
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-staging
  namespace: staging
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fastapi
      env: staging
  template:
    metadata:
      labels:
        app: fastapi
        env: staging
    spec:
      tolerations:
      - key: "node.kubernetes.io/disk-pressure"
        operator: "Exists"
        effect: "NoSchedule"
      - key: "node.kubernetes.io/memory-pressure"
        operator: "Exists"
        effect: "NoSchedule"
      containers:
      - name: fastapi
        image: guessod/datascientestapi:${DOCKER_TAG}
        imagePullPolicy: Always
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "32Mi"
            cpu: "10m"
          limits:
            memory: "64Mi"
            cpu: "50m"
---
apiVersion: v1
kind: Service
metadata:
  name: fastapi-staging
  namespace: staging
spec:
  type: NodePort
  selector:
    app: fastapi
    env: staging
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-prod
  namespace: prod
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fastapi
      env: prod
  template:
    metadata:
      labels:
        app: fastapi
        env: prod
    spec:
      tolerations:
      - key: "node.kubernetes.io/disk-pressure"
        operator: "Exists"
        effect: "NoSchedule"
      - key: "node.kubernetes.io/memory-pressure"
        operator: "Exists"
        effect: "NoSchedule"
      containers:
      - name: fastapi
        image: guessod/datascientestapi:${DOCKER_TAG}
        imagePullPolicy: Always
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "32Mi"
            cpu: "10m"
          limits:
            memory: "64Mi"
            cpu: "50m"
---
apiVersion: v1
kind: Service
metadata:
  name: fastapi-prod
  namespace: prod
spec:
  type: NodePort
  selector:
    app: fastapi
    env: prod
  ports:
  - port: 80
    targetPort: 80
EOF
          
          echo "5. Application des déploiements..."
          kubectl apply -f all-deployments.yaml --kubeconfig=$KUBECONFIG
          
          # 6. Vérification
          echo "6. Vérification des déploiements..."
          for ns in dev staging prod; do
            echo "--- Namespace: $ns ---"
            kubectl get deployments,svc -n $ns --kubeconfig=$KUBECONFIG
          done
          
          # 7. Attendre et vérifier les pods
          echo "7. Attente du démarrage des pods..."
          for attempt in {1..30}; do
            echo "Tentative $attempt/30..."
            
            # Vérifier l'état de chaque namespace
            ALL_RUNNING=true
            for ns in dev staging prod; do
              # Récupérer le nom du pod
              POD_NAME=$(kubectl get pods -n $ns -l app=fastapi -o jsonpath='{.items[0].metadata.name}' --kubeconfig=$KUBECONFIG 2>/dev/null || echo "")
              
              if [ -n "$POD_NAME" ]; then
                POD_STATUS=$(kubectl get pod $POD_NAME -n $ns -o jsonpath='{.status.phase}' --kubeconfig=$KUBECONFIG 2>/dev/null || echo "NotFound")
                
                if [ "$POD_STATUS" = "Running" ]; then
                  echo "  ✓ $ns: Pod $POD_NAME en cours d'exécution"
                elif [ "$POD_STATUS" = "Pending" ] || [ "$POD_STATUS" = "ContainerCreating" ]; then
                  echo "  ⏳ $ns: Pod en attente ($POD_STATUS)"
                  ALL_RUNNING=false
                else
                  echo "  ⚠ $ns: Statut $POD_STATUS"
                  # Afficher les détails si en échec
                  if [ "$POD_STATUS" = "Failed" ] || [ "$POD_STATUS" = "Evicted" ]; then
                    kubectl describe pod $POD_NAME -n $ns --kubeconfig=$KUBECONFIG | tail -20
                  fi
                  ALL_RUNNING=false
                fi
              else
                echo "  ❌ $ns: Aucun pod trouvé"
                ALL_RUNNING=false
              fi
            done
            
            if [ "$ALL_RUNNING" = true ]; then
              echo "✓✓✓ Tous les pods sont en cours d'exécution ✓✓✓"
              break
            fi
            
            if [ $attempt -eq 30 ]; then
              echo "⚠ Certains pods ne sont pas encore prêts, continuons quand même"
            fi
            
            sleep 2
          done
          
          # 8. Affichage final détaillé
          echo "8. État final détaillé:"
          for ns in dev staging prod; do
            echo ""
            echo "=== $ns ==="
            echo "Déploiements:"
            kubectl get deployments -n $ns --kubeconfig=$KUBECONFIG
            echo ""
            echo "Services:"
            kubectl get svc -n $ns --kubeconfig=$KUBECONFIG
            echo ""
            echo "Pods:"
            kubectl get pods -n $ns --kubeconfig=$KUBECONFIG -o wide
            echo ""
            
            # Afficher les NodePorts attribués
            NODE_PORT=$(kubectl get svc -n $ns -l app=fastapi -o jsonpath='{.items[0].spec.ports[0].nodePort}' --kubeconfig=$KUBECONFIG 2>/dev/null || echo "N/A")
            CLUSTER_IP=$(kubectl get svc -n $ns -l app=fastapi -o jsonpath='{.items[0].spec.clusterIP}' --kubeconfig=$KUBECONFIG 2>/dev/null || echo "N/A")
            echo "Accès: http://<NODE_IP>:$NODE_PORT (ClusterIP: $CLUSTER_IP)"
          done
          '''
        }
      }
    }
  }
  
  post {
    always {
      script {
        sh '''
        echo "=== RAPPORT FINAL DU PIPELINE ==="
        KUBECONFIG=".kube/config"
        
        echo "1. État du nœud:"
        kubectl get nodes -o wide --kubeconfig=$KUBECONFIG 2>/dev/null || echo "Impossible de récupérer l'état des nœuds"
        echo ""
        
        echo "2. Taints actuels:"
        kubectl describe nodes --kubeconfig=$KUBECONFIG 2>/dev/null | grep -i taint || echo "Information non disponible"
        echo ""
        
        echo "3. Résumé des déploiements:"
        echo "Namespace | Déploiement | Pods (Running/Total) | NodePort"
        echo "----------|-------------|---------------------|----------"
        for ns in dev staging prod; do
          RUNNING=$(kubectl get pods -n $ns -l app=fastapi --field-selector=status.phase=Running --no-headers --kubeconfig=$KUBECONFIG 2>/dev/null | wc -l || echo "0")
          TOTAL=$(kubectl get pods -n $ns -l app=fastapi --no-headers --kubeconfig=$KUBECONFIG 2>/dev/null | wc -l || echo "0")
          NODE_PORT=$(kubectl get svc -n $ns -l app=fastapi -o jsonpath='{.items[0].spec.ports[0].nodePort}' --kubeconfig=$KUBECONFIG 2>/dev/null || echo "N/A")
          echo "$ns       | fastapi     | $RUNNING/$TOTAL           | $NODE_PORT"
        done
        echo ""
        
        echo "4. Espace disque restant:"
        df -h / | tail -1
        echo ""
        
        echo "5. Image Docker utilisée:"
        echo "   guessod/datascientestapi:${DOCKER_TAG}"
        '''
      }
    }
    success {
      script {
        sh '''
        echo ""
        echo "✅✅✅ PIPELINE TERMINÉ AVEC SUCCÈS ✅✅✅"
        echo ""
        echo "RÉCAPITULATIF:"
        echo "1. Image Docker construite et publiée avec succès"
        echo "2. Applications déployées sur Kubernetes (dev, staging, prod)"
        echo "3. Configuration adaptée pour gérer la pression disque"
        echo ""
        echo "NEXT STEPS:"
        echo "1. Récupérer l'adresse IP de votre nœud:"
        echo "   kubectl get nodes -o wide"
        echo ""
        echo "2. Tester l'accès aux applications:"
        echo "   curl http://<NODE_IP>:<NODE_PORT_DEV>"
        echo "   curl http://<NODE_IP>:<NODE_PORT_STAGING>"
        echo "   curl http://<NODE_IP>:<NODE_PORT_PROD>"
        echo ""
        echo "3. Pour libérer durablement de l'espace disque:"
        echo "   - Nettoyer régulièrement: docker system prune -a -f"
        echo "   - Étendre la partition si possible"
        echo "   - Configurer des limites de stockage"
        '''
      }
    }
    failure {
      script {
        sh '''
        echo ""
        echo "❌❌❌ PIPELINE EN ÉCHEC ❌❌❌"
        echo ""
        echo "DÉBOGAGE RAPIDE:"
        KUBECONFIG=".kube/config"
        
        echo "1. Vérifier l'état des pods:"
        kubectl get pods -A --kubeconfig=$KUBECONFIG 2>/dev/null || true
        echo ""
        
        echo "2. Vérifier les événements récents:"
        kubectl get events -A --sort-by='.lastTimestamp' --kubeconfig=$KUBECONFIG 2>/dev/null | tail -5 || true
        echo ""
        
        echo "3. Vérifier l'espace disque:"
        df -h / | grep -v Filesystem
        echo ""
        
        echo "SOLUTIONS:"
        echo "1. Si espace disque insuffisant (<2GB libre):"
        echo "   docker system prune -a -f"
        echo "   sudo journalctl --vacuum-time=1d"
        echo ""
        echo "2. Si conflit de ports:"
        echo "   kubectl delete svc --all -n dev staging prod"
        echo ""
        echo "3. Relancer le pipeline après nettoyage"
        '''
      }
    }
  }
}
