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

    stage('Install kubectl') {
      steps {
        sh '''
          echo "Installing kubectl for correct architecture..."

          ARCH=$(uname -m)
          case "$ARCH" in
              x86_64)   KARCH=amd64 ;;
              aarch64)  KARCH=arm64 ;;
              armv7l)   KARCH=armv7 ;;
              *)        echo "Unsupported architecture: $ARCH" && exit 1 ;;
          esac

          echo "Detected architecture: $KARCH"
          VERSION=$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)

          curl -LO https://storage.googleapis.com/kubernetes-release/release/${VERSION}/bin/linux/${KARCH}/kubectl
          chmod +x kubectl
          mv kubectl $WORKSPACE/bin/
          kubectl version --client
        '''
      }
    }

    stage('Verify Cluster Access') {
      steps {
        withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG_FILE')]) {
          sh '''
            echo "Testing connection to Kubernetes cluster..."
            
            mkdir -p $WORKSPACE/.kube
            cp "$KUBECONFIG_FILE" $WORKSPACE/.kube/config
            chmod 600 $WORKSPACE/.kube/config
            export KUBECONFIG=$WORKSPACE/.kube/config

            echo "Current cluster info:"
            kubectl cluster-info

            echo "\nCurrent namespaces:"
            kubectl get ns

            echo "\nPods in the githubservices namespace:"
            kubectl get pods -n githubservices || true
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
  } 

  post {
    success {
      echo "✅ BUILD SUCCESSFUL"
    }
    failure {
      echo "❌ BUILD FAILED"
    }
    always {
      echo "Cleaning up workspace..."
      cleanWs()
    }
  }
}
