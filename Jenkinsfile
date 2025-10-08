pipeline {
  agent any

  environment {
    IMAGE_NAME = "tinotendaalfaneti/atarnet-homelab" // Docker Hub repo
    TAG = "latest"
    KUBECONFIG_CRED = 'kubeconfigglobal'
    PATH = "$WORKSPACE/bin:$PATH"
  }

  stages {
    stage('Prepare Workspace') {
        steps {
            echo "🧹 Cleaning workspace..."
            cleanWs()
            sh 'mkdir -p $WORKSPACE/bin'
        }
    }

    stage('Install kubectl') {
        steps {
            sh '''
                echo "📦 Installing kubectl for correct architecture..."

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
                    echo "🔍 Testing connection to Kubernetes cluster..."
                    
                    # Use a workspace-local kubeconfig instead of ~/.kube
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
    stage('Build & Push with Kaniko') {
      steps {
        script {
          sh '''
          echo "🚀 Building and pushing image with Kaniko..."

          cat <<EOF > kaniko.yaml
          apiVersion: v1
          kind: Pod
          metadata:
            name: kaniko-build
          spec:
            serviceAccountName: kaniko-builder
            restartPolicy: Never
            containers:
              - name: kaniko
                image: gcr.io/kaniko-project/executor:latest
                args:
                  - "--dockerfile=Dockerfile"
                  - "--context=git://github.com/tinotenda-alfaneti/atarnet-homelab.git"
                  - "--destination=${IMAGE_NAME}:${TAG}"
                  - "--cleanup"
                volumeMounts:
                  - name: docker-config
                    mountPath: /kaniko/.docker/
            volumes:
              - name: docker-config
                projected:
                  sources:
                    - secret:
                        name: dockerhub-creds
                        items:
                          - key: .dockerconfigjson
                            path: config.json
          EOF

          kubectl apply -f kaniko.yaml -n githubservices
          kubectl wait --for=condition=complete pod/kaniko-build -n githubservices --timeout=10m
          kubectl logs pod/kaniko-build -n githubservices
          kubectl delete pod/kaniko-build -n githubservices
          '''
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
            echo "🧽 Cleaning up workspace..."
            cleanWs()
        }
    }
  }
}
