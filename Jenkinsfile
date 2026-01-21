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
    
    stage('Deploiement en dev') {
      environment {
        KUBECONFIG = credentials("config") // we retrieve kubeconfig from secret file called config saved on jenkins
      }
      steps {
        script {
          sh '''
          rm -Rf .kube
          mkdir .kube
          cat $KUBECONFIG > .kube/config
          cp fastapi/values.yaml values.yml
          sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
          helm upgrade --install app fastapi --values=values.yml --namespace dev --kube-insecure-skip-tls-verify
          '''
        }
      }
    }
    
    stage('Deploiement en staging') {
      environment {
        KUBECONFIG = credentials("config") // we retrieve kubeconfig from secret file called config saved on jenkins
      }
      steps {
        script {
          sh '''
          rm -Rf .kube
          mkdir .kube
          cat $KUBECONFIG > .kube/config
          cp fastapi/values.yaml values.yml
          sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
          helm upgrade --install app fastapi --values=values.yml --namespace staging --kube-insecure-skip-tls-verify
          '''
        }
      }
    }
    
    stage('Deploiement en prod') {
      environment {
        KUBECONFIG = credentials("config") // we retrieve kubeconfig from secret file called config saved on jenkins
      }
      steps {
        // Create an Approval Button with a timeout of 15minutes.
        // this require a manual validation in order to deploy on production environment
        timeout(time: 15, unit: "MINUTES") {
          input message: 'Do you want to deploy in production ?', ok: 'Yes'
        }
        script {
          sh '''
          rm -Rf .kube
          mkdir .kube
          cat $KUBECONFIG > .kube/config
          cp fastapi/values.yaml values.yml
          sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
          helm upgrade --install app fastapi --values=values.yml --namespace prod --kube-insecure-skip-tls-verify
          '''
        }
      }
    }
  }
}
