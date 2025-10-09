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
          echo "📦 Installing kubectl and helm..."
          ARCH=$(uname -m)
          case "$ARCH" in
              x86_64)   KARCH=amd64 ;;
              aarch64)  KARCH=arm64 ;;
              armv7l)   KARCH=armv7 ;;
              *)        echo "Unsupported architecture: $ARCH" && exit 1 ;;
          esac

          # --- Install kubectl ---
          VERSION=$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)
          echo "Installing kubectl version ${VERSION} for ${KARCH}..."
          curl -sLO https://storage.googleapis.com/kubernetes-release/release/${VERSION}/bin/linux/${KARCH}/kubectl
          chmod +x kubectl && mv kubectl $WORKSPACE/bin/
          echo "✅ kubectl installed:"
          $WORKSPACE/bin/kubectl version --client

          # --- Install helm (manual, no sudo) ---
          HELM_VERSION="v3.14.4"
          echo "Installing Helm ${HELM_VERSION} for ${KARCH}..."
          curl -sLO https://get.helm.sh/helm-${HELM_VERSION}-linux-${KARCH}.tar.gz
          tar -zxf helm-${HELM_VERSION}-linux-${KARCH}.tar.gz
          mv linux-${KARCH}/helm $WORKSPACE/bin/helm
          chmod +x $WORKSPACE/bin/helm
          rm -rf linux-${KARCH} helm-${HELM_VERSION}-linux-${KARCH}.tar.gz
          echo "✅ helm installed:"
          $WORKSPACE/bin/helm version
        '''
      }
    }

    stage('Verify Cluster Access') {
      steps {
        withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG_FILE')]) {
          sh '''
            echo "🔐 Verifying Kubernetes access..."
            mkdir -p $WORKSPACE/.kube
            cp "$KUBECONFIG_FILE" $WORKSPACE/.kube/config
            chmod 600 $WORKSPACE/.kube/config
            export KUBECONFIG=$WORKSPACE/.kube/config
            $WORKSPACE/bin/kubectl cluster-info
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
              echo "🚀 Launching Kaniko Job..."
              $WORKSPACE/bin/kubectl delete job kaniko-job -n githubservices --ignore-not-found=true
              $WORKSPACE/bin/kubectl apply -f $WORKSPACE/k8s/kaniko.yaml -n githubservices
              $WORKSPACE/bin/kubectl wait --for=condition=complete job/kaniko-job -n githubservices --timeout=10m
              $WORKSPACE/bin/kubectl logs job/kaniko-job -n githubservices
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
              echo "🔍 Launching Trivy scan job in Kubernetes..."
              $WORKSPACE/bin/kubectl delete job trivy-scan -n githubservices --ignore-not-found=true
              sed "s|tinorodney/atarnet-homelab:latest|${IMAGE_NAME}:${TAG}|g" \
                $WORKSPACE/k8s/trivy.yaml | $WORKSPACE/bin/kubectl apply -f - -n githubservices
              $WORKSPACE/bin/kubectl wait --for=condition=complete job/trivy-scan -n githubservices --timeout=5m || true
              echo "Trivy scan logs:"
              $WORKSPACE/bin/kubectl logs job/trivy-scan -n githubservices || true
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
            echo "⚙️ Deploying with Helm..."
            $WORKSPACE/bin/helm upgrade --install atarnet-homelab $WORKSPACE/charts/atarnet-homelab \
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
            echo "🪵 Fetching logs from app..."
            POD=$($WORKSPACE/bin/kubectl get pods -n githubservices -l app=atarnet-homelab -o jsonpath="{.items[0].metadata.name}")
            $WORKSPACE/bin/kubectl logs $POD -n githubservices || true
          '''
        }
      }
    }
  }

  post {
    success {
      echo "✅ BUILD SUCCESS."
    }
    failure {
      echo "❌ BUILD FAILED."
    }
    always {
      echo "🧹 Cleaning workspace..."
      cleanWs()
    }
  }
}
