pipeline {
  agent any

  environment {

    IMAGE_NAME = "tinorodney/atarnet-homelab" // Docker Hub repo
    TAG = "latest"
    KUBECONFIG_CRED = 'kubeconfigglobal'
    PATH = "$WORKSPACE/bin:$PATH"
  }

  stages {
    stage('Checkout') {
        steps {
           checkout scm
           sh 'mkdir -p $WORKSPACE/bin'
        }
    }

    stage('Install Tools') {
      steps {
        sh '''
          echo "Installing kubectl and helm..."
          ARCH=$(uname -m)
          case "$ARCH" in
              x86_64)   KARCH=amd64 ;;
              aarch64)  KARCH=arm64 ;;
              armv7l)   KARCH=armv7 ;;
              *)        echo "Unsupported arch: $ARCH" && exit 1 ;;
          esac

          VERSION=$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)
          curl -LO https://storage.googleapis.com/kubernetes-release/release/${VERSION}/bin/linux/${KARCH}/kubectl
          chmod +x kubectl && mv kubectl $WORKSPACE/bin/
          
          curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
          helm version
        '''
      }
    }

    stage('Verify Cluster Access') {
      steps {
        withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG_FILE')]) {
          sh '''
            mkdir -p $WORKSPACE/.kube
            cp "$KUBECONFIG_FILE" $WORKSPACE/.kube/config
            chmod 600 $WORKSPACE/.kube/config
            export KUBECONFIG=$WORKSPACE/.kube/config
            kubectl cluster-info
          '''
        }
      }
    }

    stage('Debug Workspace') {
      steps {
        sh 'ls -R $WORKSPACE'
      }
    }


    stage('Build & Push with Kaniko') {
      steps {
        withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG_FILE')]) {
          script {
            sh '''
              export KUBECONFIG=$KUBECONFIG_FILE
              echo "Launching Kaniko Job..."
              kubectl delete job kaniko-job -n githubservices --ignore-not-found=true
              # Apply your Kaniko job YAML here
              kubectl apply -f $WORKSPACE/k8s/kaniko.yaml -n githubservices
              kubectl wait --for=condition=complete job/kaniko-job -n githubservices --timeout=10m
              kubectl logs job/kaniko-job -n githubservices
            '''
          }
        }
      }
    }

    stage('Scan Image with Trivy (K8s Job)') {
      steps {
        withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG_FILE')]) {
          script {
            sh '''
              export KUBECONFIG=$KUBECONFIG_FILE
              echo "Launching Trivy scan job in Kubernetes..."

              # Delete any previous scan job
              kubectl delete job trivy-scan -n githubservices --ignore-not-found=true

              # Update the image in YAML dynamically (in case tag changes)
              sed "s|tinorodney/atarnet-homelab:latest|${IMAGE_NAME}:${TAG}|g" \
                $WORKSPACE/k8s/trivy.yaml | kubectl apply -f - -n githubservices

              # Wait for completion (up to 5 minutes)
              kubectl wait --for=condition=complete job/trivy-scan -n githubservices --timeout=5m || true

              echo "Trivy scan logs:"
              kubectl logs job/trivy-scan -n githubservices || true
            '''
          }
        }
      }
    }


    stage('Deploy with Helm') {
      steps {
        withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG_FILE')]) {
          sh '''
            export KUBECONFIG=$KUBECONFIG_FILE
            echo "Deploying with Helm..."
            helm upgrade --install atarnet-homelab $WORKSPACE/charts/atarnet-homelab \
              --namespace githubservices \
              --create-namespace \
              --set image.repository=${IMAGE_NAME} \
              --set image.tag=${TAG}
          '''
        }
      }
    }

    stage('Fetch App Logs') {
      steps {
        withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG_FILE')]) {
          sh '''
            export KUBECONFIG=$KUBECONFIG_FILE
            echo "Fetching logs..."
            POD=$(kubectl get pods -n githubservices -l app=atarnet-homelab -o jsonpath="{.items[0].metadata.name}")
            kubectl logs $POD -n githubservices || true
          '''
        }
      }
    }
  }

  post {
    success {
      echo "✅BUILD SUCCESS."
    }
    failure {
      echo "❌BUILD FAILED."
    }
    always {
      cleanWs()
    }
  }
}
