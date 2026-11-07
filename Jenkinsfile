pipeline {
  agent {
    kubernetes {
      yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: maven
      image: maven:3.9-eclipse-temurin-17
      command: [cat]
      tty: true
      resources:
        requests:
          memory: 512Mi
        limits:
          memory: 1536Mi
    - name: kaniko
      image: gcr.io/kaniko-project/executor:debug
      command: [/busybox/cat]
      tty: true
    - name: git
      image: alpine/git:latest
      command: [cat]
      tty: true
'''
    }
  }

  environment {
    IMAGE = 'lizakalinichenko/spring-petclinic'
  }

  stages {
    stage('Prepare tag') {
      steps {
        script {
          env.TAG = "${env.BUILD_NUMBER}-${(env.GIT_COMMIT ?: 'local').take(7)}"
        }
        echo "Image tag: ${env.TAG}"
      }
    }

    stage('Test') {
      steps {
        container('maven') {
          sh 'mvn -B test'
        }
      }
    }

    stage('Build') {
      steps {
        container('maven') {
          sh 'mvn -B package -DskipTests'
        }
      }
    }

    stage('Image build and push') {
      steps {
        container('kaniko') {
          withCredentials([usernamePassword(credentialsId: 'dockerhub',
                                            usernameVariable: 'DH_USER',
                                            passwordVariable: 'DH_PASS')]) {
            sh '''
AUTH=$(printf "%s:%s" "$DH_USER" "$DH_PASS" | base64 | tr -d '\\n')
mkdir -p /kaniko/.docker
cat > /kaniko/.docker/config.json <<EOF
{"auths":{"https://index.docker.io/v1/":{"auth":"$AUTH"}}}
EOF
/kaniko/executor \
  --context "dir://$WORKSPACE" \
  --dockerfile "$WORKSPACE/Dockerfile" \
  --destination "$IMAGE:$TAG" \
  --destination "$IMAGE:latest"
'''
          }
        }
      }
    }

    stage('Update Helm values') {
      steps {
        container('git') {
          withCredentials([usernamePassword(credentialsId: 'github-gitops',
                                            usernameVariable: 'GH_USER',
                                            passwordVariable: 'GH_PASS')]) {
            sh '''
git config --global user.email "jenkins@localhost"
git config --global user.name "jenkins"
rm -rf gitops
git clone "https://$GH_USER:$GH_PASS@github.com/l-lisssa/1petclinic-gitops.git" gitops
cd gitops
sed -i "s|tag: .*|tag: \\"$TAG\\"|" charts/petclinic/values.yaml
git add charts/petclinic/values.yaml
git commit -m "ci: petclinic image tag $TAG"
git push
'''
          }
        }
      }
    }
  }
}
