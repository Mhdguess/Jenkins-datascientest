pipeline {
  environment {
    DOCKER_ID = "guessod"
    DOCKER_IMAGE = "datascientestapi"
    DOCKER_TAG = "v.${BUILD_ID}.0"
  }
  agent any
  
  // AJOUT : Configuration du polling pour déclenchement automatique
  triggers {
    pollSCM('* * * * *')  // Vérifie les changements toutes les minutes
  }
  
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
    
    stage('Déploiement DEV') {
      steps {
        script {
          sh '''
          echo "=== DÉPLOIEMENT ENVIRONNEMENT DEV ==="
          mkdir -p .kube
          cp /var/lib/jenkins/.kube/config .kube/config
          KUBECONFIG=".kube/config"
          
          # 1. Supprimer le taint disk-pressure
          echo "1. Suppression du taint disk-pressure..."
          kubectl taint nodes --all node.kubernetes.io/disk-pressure:NoSchedule- --kubeconfig=$KUBECONFIG 2>/dev/null || true
          
          # 2. NETTOYAGE COMPLET - Supprimer tous les déploiements et services existants EN DEV
          echo "2. Nettoyage des ressources DEV..."
          echo "--- Nettoyage namespace dev ---"
          kubectl delete deployment fastapi-dev -n dev --ignore-not-found=true --kubeconfig=$KUBECONFIG
          kubectl delete deployment app-fastapi -n dev --ignore-not-found=true --kubeconfig=$KUBECONFIG
          kubectl delete svc fastapi-dev -n dev --ignore-not-found=true --kubeconfig=$KUBECONFIG
          kubectl delete svc app-fastapi -n dev --ignore-not-found=true --kubeconfig=$KUBECONFIG
          kubectl delete pods --all -n dev --kubeconfig=$KUBECONFIG --ignore-not-found=true
          
          # Attendre que tout soit supprimé
          sleep 5
          
          # 3. Créer le namespace s'il n'existe pas
          echo "3. Création du namespace dev..."
          kubectl create namespace dev --dry-run=client -o yaml --kubeconfig=$KUBECONFIG | kubectl apply -f - --kubeconfig=$KUBECONFIG
          
          # 4. Créer le déploiement DEV
          echo "4. Création du déploiement DEV..."
          
          cat > deployment-dev.yaml << EOF
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
EOF
          
          echo "5. Application du déploiement DEV..."
          kubectl apply -f deployment-dev.yaml --kubeconfig=$KUBECONFIG
          
          # 6. Vérification DEV
          echo "6. Vérification du déploiement DEV..."
          echo "--- Namespace: dev ---"
          kubectl get deployments,svc -n dev --kubeconfig=$KUBECONFIG
          
          # 7. Attendre et vérifier le pod DEV
          echo "7. Attente du démarrage du pod DEV..."
          for attempt in {1..30}; do
            POD_NAME=$(kubectl get pods -n dev -l app=fastapi,env=dev -o jsonpath='{.items[0].metadata.name}' --kubeconfig=$KUBECONFIG 2>/dev/null || echo "")
            
            if [ -n "$POD_NAME" ]; then
              POD_STATUS=$(kubectl get pod $POD_NAME -n dev -o jsonpath='{.status.phase}' --kubeconfig=$KUBECONFIG 2>/dev/null || echo "NotFound")
              
              if [ "$POD_STATUS" = "Running" ]; then
                echo "  ✓ DEV: Pod $POD_NAME en cours d'exécution"
                break
              elif [ "$POD_STATUS" = "Pending" ] || [ "$POD_STATUS" = "ContainerCreating" ]; then
                echo "  ⏳ DEV: Pod en attente ($POD_STATUS) - tentative $attempt/30"
              else
                echo "  ⚠ DEV: Statut $POD_STATUS"
              fi
            else
              echo "  ❌ DEV: Aucun pod trouvé - tentative $attempt/30"
            fi
            
            if [ $attempt -eq 30 ]; then
              echo "⚠ Pod DEV n'est pas encore prêt, continuons quand même"
            fi
            
            sleep 2
          done
          
          # 8. Affichage DEV
          echo "8. État DEV:"
          echo "=== dev ==="
          echo "Déploiements:"
          kubectl get deployments -n dev --kubeconfig=$KUBECONFIG
          echo ""
          echo "Services:"
          kubectl get svc -n dev --kubeconfig=$KUBECONFIG
          echo ""
          echo "Pods:"
          kubectl get pods -n dev --kubeconfig=$KUBECONFIG -o wide
          echo ""
          
          NODE_PORT=$(kubectl get svc fastapi-dev -n dev -o jsonpath='{.spec.ports[0].nodePort}' --kubeconfig=$KUBECONFIG 2>/dev/null || echo "N/A")
          CLUSTER_IP=$(kubectl get svc fastapi-dev -n dev -o jsonpath='{.spec.clusterIP}' --kubeconfig=$KUBECONFIG 2>/dev/null || echo "N/A")
          echo "Accès DEV: http://<NODE_IP>:$NODE_PORT (ClusterIP: $CLUSTER_IP)"
          '''
        }
      }
    }
    
    stage('Tests DEV') {
      steps {
        script {
          sh '''
          echo "=== TESTS ENVIRONNEMENT DEV ==="
          KUBECONFIG=".kube/config"
          
          # Récupérer les infos pour tester
          NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}' --kubeconfig=$KUBECONFIG 2>/dev/null || echo "localhost")
          NODE_PORT=$(kubectl get svc fastapi-dev -n dev -o jsonpath='{.spec.ports[0].nodePort}' --kubeconfig=$KUBECONFIG 2>/dev/null || echo "")
          
          if [ -n "$NODE_PORT" ]; then
            echo "Test de l'application DEV sur $NODE_IP:$NODE_PORT..."
            for i in {1..5}; do
              if curl -s -f http://$NODE_IP:$NODE_PORT > /dev/null; then
                echo "✓ Test DEV réussi à la tentative $i"
                curl http://$NODE_IP:$NODE_PORT
                break
              else
                echo "Tentative $i/5 échouée, attente de 2 secondes..."
                sleep 2
              fi
            done
          else
            echo "⚠ Impossible de récupérer le NodePort DEV"
          fi
          '''
        }
      }
    }
    
    stage('Déploiement STAGING') {
      steps {
        script {
          sh '''
          echo "=== DÉPLOIEMENT ENVIRONNEMENT STAGING ==="
          KUBECONFIG=".kube/config"
          
          # 1. NETTOYAGE COMPLET - Supprimer tous les déploiements et services existants EN STAGING
          echo "1. Nettoyage des ressources STAGING..."
          echo "--- Nettoyage namespace staging ---"
          kubectl delete deployment fastapi-staging -n staging --ignore-not-found=true --kubeconfig=$KUBECONFIG
          kubectl delete deployment app-fastapi -n staging --ignore-not-found=true --kubeconfig=$KUBECONFIG
          kubectl delete svc fastapi-staging -n staging --ignore-not-found=true --kubeconfig=$KUBECONFIG
          kubectl delete svc app-fastapi -n staging --ignore-not-found=true --kubeconfig=$KUBECONFIG
          kubectl delete pods --all -n staging --kubeconfig=$KUBECONFIG --ignore-not-found=true
          
          sleep 3
          
          # 2. Créer le namespace s'il n'existe pas
          echo "2. Création du namespace staging..."
          kubectl create namespace staging --dry-run=client -o yaml --kubeconfig=$KUBECONFIG | kubectl apply -f - --kubeconfig=$KUBECONFIG
          
          # 3. Créer le déploiement STAGING
          echo "3. Création du déploiement STAGING..."
          
          cat > deployment-staging.yaml << EOF
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
EOF
          
          echo "4. Application du déploiement STAGING..."
          kubectl apply -f deployment-staging.yaml --kubeconfig=$KUBECONFIG
          
          # 5. Vérification STAGING
          echo "5. Vérification du déploiement STAGING..."
          echo "--- Namespace: staging ---"
          kubectl get deployments,svc -n staging --kubeconfig=$KUBECONFIG
          
          # 6. Attendre et vérifier le pod STAGING
          echo "6. Attente du démarrage du pod STAGING..."
          for attempt in {1..30}; do
            POD_NAME=$(kubectl get pods -n staging -l app=fastapi,env=staging -o jsonpath='{.items[0].metadata.name}' --kubeconfig=$KUBECONFIG 2>/dev/null || echo "")
            
            if [ -n "$POD_NAME" ]; then
              POD_STATUS=$(kubectl get pod $POD_NAME -n staging -o jsonpath='{.status.phase}' --kubeconfig=$KUBECONFIG 2>/dev/null || echo "NotFound")
              
              if [ "$POD_STATUS" = "Running" ]; then
                echo "  ✓ STAGING: Pod $POD_NAME en cours d'exécution"
                break
              elif [ "$POD_STATUS" = "Pending" ] || [ "$POD_STATUS" = "ContainerCreating" ]; then
                echo "  ⏳ STAGING: Pod en attente ($POD_STATUS) - tentative $attempt/30"
              else
                echo "  ⚠ STAGING: Statut $POD_STATUS"
              fi
            else
              echo "  ❌ STAGING: Aucun pod trouvé - tentative $attempt/30"
            fi
            
            if [ $attempt -eq 30 ]; then
              echo "⚠ Pod STAGING n'est pas encore prêt, continuons quand même"
            fi
            
            sleep 2
          done
          
          # 7. Affichage STAGING
          echo "7. État STAGING:"
          echo "=== staging ==="
          echo "Déploiements:"
          kubectl get deployments -n staging --kubeconfig=$KUBECONFIG
          echo ""
          echo "Services:"
          kubectl get svc -n staging --kubeconfig=$KUBECONFIG
          echo ""
          echo "Pods:"
          kubectl get pods -n staging --kubeconfig=$KUBECONFIG -o wide
          echo ""
          
          NODE_PORT=$(kubectl get svc fastapi-staging -n staging -o jsonpath='{.spec.ports[0].nodePort}' --kubeconfig=$KUBECONFIG 2>/dev/null || echo "N/A")
          CLUSTER_IP=$(kubectl get svc fastapi-staging -n staging -o jsonpath='{.spec.clusterIP}' --kubeconfig=$KUBECONFIG 2>/dev/null || echo "N/A")
          echo "Accès STAGING: http://<NODE_IP>:$NODE_PORT (ClusterIP: $CLUSTER_IP)"
          '''
        }
      }
    }
    
    stage('Tests STAGING') {
      steps {
        script {
          sh '''
          echo "=== TESTS ENVIRONNEMENT STAGING ==="
          KUBECONFIG=".kube/config"
          
          # Récupérer les infos pour tester
          NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}' --kubeconfig=$KUBECONFIG 2>/dev/null || echo "localhost")
          NODE_PORT=$(kubectl get svc fastapi-staging -n staging -o jsonpath='{.spec.ports[0].nodePort}' --kubeconfig=$KUBECONFIG 2>/dev/null || echo "")
          
          if [ -n "$NODE_PORT" ]; then
            echo "Test de l'application STAGING sur $NODE_IP:$NODE_PORT..."
            for i in {1..5}; do
              if curl -s -f http://$NODE_IP:$NODE_PORT > /dev/null; then
                echo "✓ Test STAGING réussi à la tentative $i"
                curl http://$NODE_IP:$NODE_PORT
                break
              else
                echo "Tentative $i/5 échouée, attente de 2 secondes..."
                sleep 2
              fi
            done
          else
            echo "⚠ Impossible de récupérer le NodePort STAGING"
          fi
          '''
        }
      }
    }
    
    // ÉTAPE DE VALIDATION MANUELLE POUR LA PRODUCTION
    stage('Validation pour PRODUCTION') {
      steps {
        timeout(time: 15, unit: 'MINUTES') {
          input message: 'Voulez-vous déployer en PRODUCTION ?', ok: 'Oui, déployer en PRODUCTION'
        }
      }
    }
    
    stage('Déploiement PRODUCTION') {
      steps {
        script {
          sh '''
          echo "=== DÉPLOIEMENT ENVIRONNEMENT PRODUCTION ==="
          KUBECONFIG=".kube/config"
          
          # 1. NETTOYAGE COMPLET - Supprimer tous les déploiements et services existants EN PROD
          echo "1. Nettoyage des ressources PRODUCTION..."
          echo "--- Nettoyage namespace prod ---"
          kubectl delete deployment fastapi-prod -n prod --ignore-not-found=true --kubeconfig=$KUBECONFIG
          kubectl delete deployment app-fastapi -n prod --ignore-not-found=true --kubeconfig=$KUBECONFIG
          kubectl delete svc fastapi-prod -n prod --ignore-not-found=true --kubeconfig=$KUBECONFIG
          kubectl delete svc app-fastapi -n prod --ignore-not-found=true --kubeconfig=$KUBECONFIG
          kubectl delete pods --all -n prod --kubeconfig=$KUBECONFIG --ignore-not-found=true
          
          sleep 3
          
          # 2. Créer le namespace s'il n'existe pas
          echo "2. Création du namespace prod..."
          kubectl create namespace prod --dry-run=client -o yaml --kubeconfig=$KUBECONFIG | kubectl apply -f - --kubeconfig=$KUBECONFIG
          
          # 3. Créer le déploiement PRODUCTION
          echo "3. Création du déploiement PRODUCTION..."
          
          cat > deployment-prod.yaml << EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-prod
  namespace: prod
spec:
  replicas: 2  # Augmenter à 2 réplicas en production
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
            memory: "64Mi"  # Augmenter les ressources en production
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
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
          
          echo "4. Application du déploiement PRODUCTION..."
          kubectl apply -f deployment-prod.yaml --kubeconfig=$KUBECONFIG
          
          # 5. Vérification PRODUCTION
          echo "5. Vérification du déploiement PRODUCTION..."
          echo "--- Namespace: prod ---"
          kubectl get deployments,svc -n prod --kubeconfig=$KUBECONFIG
          
          # 6. Attendre et vérifier les pods PRODUCTION
          echo "6. Attente du démarrage des pods PRODUCTION..."
          for attempt in {1..40}; do  # Plus de temps pour la production
            POD_COUNT=$(kubectl get pods -n prod -l app=fastapi,env=prod --no-headers --kubeconfig=$KUBECONFIG 2>/dev/null | wc -l || echo "0")
            RUNNING_COUNT=$(kubectl get pods -n prod -l app=fastapi,env=prod --field-selector=status.phase=Running --no-headers --kubeconfig=$KUBECONFIG 2>/dev/null | wc -l || echo "0")
            
            if [ "$POD_COUNT" -eq 2 ] && [ "$RUNNING_COUNT" -eq 2 ]; then
              echo "  ✓ PRODUCTION: 2/2 pods en cours d'exécution"
              break
            else
              echo "  ⏳ PRODUCTION: $RUNNING_COUNT/$POD_COUNT pods prêts - tentative $attempt/40"
            fi
            
            if [ $attempt -eq 40 ]; then
              echo "⚠ Pods PRODUCTION pas tous prêts, continuons quand même"
            fi
            
            sleep 2
          done
          
          # 7. Affichage PRODUCTION
          echo "7. État PRODUCTION:"
          echo "=== prod ==="
          echo "Déploiements:"
          kubectl get deployments -n prod --kubeconfig=$KUBECONFIG
          echo ""
          echo "Services:"
          kubectl get svc -n prod --kubeconfig=$KUBECONFIG
          echo ""
          echo "Pods:"
          kubectl get pods -n prod --kubeconfig=$KUBECONFIG -o wide
          echo ""
          
          NODE_PORT=$(kubectl get svc fastapi-prod -n prod -o jsonpath='{.spec.ports[0].nodePort}' --kubeconfig=$KUBECONFIG 2>/dev/null || echo "N/A")
          CLUSTER_IP=$(kubectl get svc fastapi-prod -n prod -o jsonpath='{.spec.clusterIP}' --kubeconfig=$KUBECONFIG 2>/dev/null || echo "N/A")
          echo "Accès PRODUCTION: http://<NODE_IP>:$NODE_PORT (ClusterIP: $CLUSTER_IP)"
          '''
        }
      }
    }
    
    stage('Tests PRODUCTION') {
      steps {
        script {
          sh '''
          echo "=== TESTS ENVIRONNEMENT PRODUCTION ==="
          KUBECONFIG=".kube/config"
          
          # Récupérer les infos pour tester
          NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}' --kubeconfig=$KUBECONFIG 2>/dev/null || echo "localhost")
          NODE_PORT=$(kubectl get svc fastapi-prod -n prod -o jsonpath='{.spec.ports[0].nodePort}' --kubeconfig=$KUBECONFIG 2>/dev/null || echo "")
          
          if [ -n "$NODE_PORT" ]; then
            echo "Test de l'application PRODUCTION sur $NODE_IP:$NODE_PORT..."
            for i in {1..5}; do
              if curl -s -f http://$NODE_IP:$NODE_PORT > /dev/null; then
                echo "✓ Test PRODUCTION réussi à la tentative $i"
                curl http://$NODE_IP:$NODE_PORT
                break
              else
                echo "Tentative $i/5 échouée, attente de 2 secondes..."
                sleep 2
              fi
            done
          else
            echo "⚠ Impossible de récupérer le NodePort PRODUCTION"
          fi
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
        echo "2. Applications déployées sur Kubernetes"
        echo "3. Déploiement progressif avec validation manuelle pour la production"
        echo ""
        echo "NEXT STEPS:"
        echo "1. Récupérer l'adresse IP de votre nœud:"
        echo "   kubectl get nodes -o wide"
        echo ""
        echo "2. Tester l'accès aux applications:"
        for ns in dev staging prod; do
          NODE_PORT=$(kubectl get svc fastapi-$ns -n $ns -o jsonpath='{.spec.ports[0].nodePort}' 2>/dev/null || echo "N/A")
          if [ "$NODE_PORT" != "N/A" ]; then
            echo "   $ns: curl http://<NODE_IP>:$NODE_PORT"
          fi
        done
        echo ""
        echo "3. Pour la production: surveiller les logs et métriques"
        echo "   kubectl logs -n prod -l app=fastapi --tail=10"
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
      echo "This will run if the job failed"
      mail to: "mohamedguessod@gmail.com",
           subject: "${env.JOB_NAME} - Build # ${env.BUILD_ID} has failed",
           body: "For more info on the pipeline failure, check out the console output at ${env.BUILD_URL}"
    }
  }
}
