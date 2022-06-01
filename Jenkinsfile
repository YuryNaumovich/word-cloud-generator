pipeline {
    environment {
    NEXUS_IP = '192.168.56.101'
    STAGE_IP = 'localhost'
    NEXUS_REPOSITORY = 'word-cloud-build'
    NEXUS_DOWNLOAD_USER = 'browse'
    NEXUS_DOWNLOAD_PASSWORD = 'browse'
    ARTIFACT_ID = 'word-cloud-generator'
    NEXUS_CREDENTIALSID = '213dd042-007a-4878-8b63-890c258cc055'
    INSTALL = ''' #service wordcloud stop
                  curl -u ${NEXUS_DOWNLOAD_USER}:${NEXUS_DOWNLOAD_PASSWORD} -X GET "http://${NEXUS_IP}:8081/repository/${NEXUS_REPOSITORY}/1/${ARTIFACT_ID}/1.$BUILD_NUMBER/${ARTIFACT_ID}-1.$BUILD_NUMBER.gz" -o /opt/wordcloud/${ARTIFACT_ID}.gz
                  gunzip -f /opt/wordcloud/${ARTIFACT_ID}.gz 
                  chmod +x /opt/wordcloud/${ARTIFACT_ID}
                  /opt/wordcloud/word-cloud-generator & '''
    TEST = '''res=`curl -s -H "Content-Type: application/json" -d '{"text":"ths is a really really really important thing this is"}' http://${STAGE_IP}:8888/version | jq '. | length'`
              if [ "1" != "$res" ]; then
                  exit 99
              fi
              res=`curl -s -H "Content-Type: application/json" -d '{"text":"ths is a really really really important thing this is"}' http://${STAGE_IP}:8888/api | jq '. | length'`
              if [ "7" != "$res" ]; then
                  exit 99
              fi'''
    }
    
    agent any
    
    stages {

        stage('Build') {
            agent { dockerfile { filename 'Dockerfile'} }
            steps {
                git url: 'https://github.com/Fenikks/word-cloud-generator'
                sh '''go version
                      make lint
                      make test
                      export GOPATH=$WORKSPACE/go
                      export PATH="$PATH:$(go env GOPATH)/bin"
                      export GOROOT="/usr/local/go"
                      go get github.com/tools/godep
                      go get github.com/smartystreets/goconvey
                      go get github.com/GeertJohan/go.rice/rice
                      go get github.com/wickett/word-cloud-generator/wordyapi
                      go get github.com/gorilla/mux
                      rm -f ./artifacts/word-cloud-generator
                      sed -i "s/1.DEVELOPMENT/1.$BUILD_NUMBER/g" static/version
                      GOOS=linux GOARCH=amd64 go build -o ./artifacts/${ARTIFACT_ID} -v .
                      gzip -f artifacts/${ARTIFACT_ID}
                      mv artifacts/${ARTIFACT_ID}.gz artifacts/${ARTIFACT_ID}
                      ls -l artifacts/'''
                nexusArtifactUploader (
                    artifacts: 
                    [[artifactId: "${ARTIFACT_ID}", 
                    classifier: '', file: "artifacts/${ARTIFACT_ID}",
                    type: 'gz']],
                     credentialsId: "${NEXUS_CREDENTIALSID}", 
                     groupId: '1', nexusUrl: "${NEXUS_IP}:8081", 
                     nexusVersion: 'nexus3', protocol: 'http', 
                     repository: "${NEXUS_REPOSITORY}", version: '1.$BUILD_NUMBER'
                )
            }

        }
        
        
        stage('Install on staging server') {
            agent { dockerfile { filename 'alpine/Dockerfile'
                                 args '--network vagrant_vagrant' } }
            steps {
                sh " ${INSTALL} "
                sh "${TEST}"
            }
        }
    }
}
