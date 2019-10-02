
pipeline {
  agent {
    label 'master'
  }
  environment {
    VERSION="0.1"
    APP_NAME = "front-end"
    TAG = "neotysdevopsdemo/${APP_NAME}"
    TAG_DEV = "${TAG}:DEV-${VERSION}"
    TAG_STAGING = "${TAG}-stagging:${VERSION}"
    GROUP="neotysdevopsdemowithvirtualize"
    VIRTUALIZE_PROXY="soavirt:9080/para_catalogue"
    VIRTUALIZE_SERVICE="soavirt:9080/para_catalogue"
    //VIRTUALIZE_CART_PROXY="carts"
   // VIRTUALIZE_CART_SERVICE="carts%s"
    VIRTUALIZE_CART_PROXY="soavirt:9080/para_cart"
    VIRTUALIZE_CART_SERVICE="soavirt:9080/para_cart"
    //VIRTUALIZE_PROXY="catalogue"
    //VIRTUALIZE_SERVICE="catalogue%s"
    ARTEFACT_ID = "sockshop-" + "${env.APP_NAME}"
    DYNATRACEID="${env.DT_ACCOUNTID}.live.dynatrace.com"
    DYNATRACEAPIKEY="${env.DT_API_TOKEN}"
    NLAPIKEY="${env.NL_WEB_API_KEY}"
    NL_DT_TAG="app:${env.APP_NAME},environment:staging"
    OUTPUTSANITYCHECK="$WORKSPACE/infrastructure/sanitycheck.json"
    NEOLOAD_ASCODEFILE="$WORKSPACE/test/neoload/Frontend_neoload.yaml"
    NEOLOAD_ANOMALIEDETECTIONFILE="$WORKSPACE/monspec/front-end_monspec.json"
    BASICCHECKURI="health"
    TAGURI="tags"
    COMMIT = "DEV-${VERSION}"
    HOST="ec2-34-249-248-141.eu-west-1.compute.amazonaws.com"
  }
  stages {

    /*stage('build app')
    {
      agent {
            docker { image 'heliostech/jenkins-slave-nodejs:latest'
              reuseNode true
              }
        }
        steps{
            sh 'npm install'
        }
    }*/
    stage('Docker build') {

      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerHub', passwordVariable: 'TOKEN', usernameVariable: 'USER')]) {
          sh "sed -i 's,TOREPLACE_VIRTUALIZE_CART_SERVICE,${VIRTUALIZE_CART_PROXY},'  $WORKSPACE/test/api/endpoints_test.js"
          sh "sed -i 's,TOREPLACE_VIRTUALIZE_CART_SERVICE,${VIRTUALIZE_CART_SERVICE},'  $WORKSPACE/api/endpoints.js"
          sh "sed -i 's,TOREPLACE_VIRTUALIZE,${VIRTUALIZE_PROXY},'  $WORKSPACE/test/api/endpoints_test.js"
          sh "sed -i 's,TOREPLACE_VIRTUALIZE,${VIRTUALIZE_SERVICE},'  $WORKSPACE/api/endpoints.js"
          sh "docker build -t ${GROUP}/${APP_NAME}:DEV-0.1 ."
          sh "docker tag ${GROUP}/${APP_NAME}:DEV-0.1 ${TAG_DEV}"
          sh "docker login --username=${USER} --password=${TOKEN}"
          sh "docker push ${TAG_DEV}"
          sh "docker tag ${TAG_DEV} ${TAG_STAGING}"
          sh "docker push ${TAG_STAGING}"
        }

      }
    }


    stage('Deploy to dev namespace') {
        steps {
          sh "sed -i 's,VERSION_TO_REPLACE,${VERSION},'  $WORKSPACE/docker-compose.yml"
          sh 'docker-compose -f $WORKSPACE/docker-compose.yml up -d'

        }

    }
     stage('Launch Selenium script') {

                  steps {
                      sh 'df -h'
                       sleep 90
                  }

              }
    stage('Start NeoLoad infrastructure') {

              steps {
                  sh 'docker-compose -f $WORKSPACE/infrastructure/infrastructure/neoload/lg/docker-compose.yml up -d'

              }

          }

     stage('Run load test') {
            agent {
                dockerfile {
                    args '--user root -v /tmp:/tmp --network=parasoft'
                    dir 'infrastructure/infrastructure/neoload/controller/'
                }
            }
            steps {

                     sh "sed -i 's/HOST_TO_REPLACE/${HOST}/'  $WORKSPACE/test/neoload/Frontend_neoload.yaml"
                     sh "sed -i 's/PORT_TO_REPLACE/80/' $WORKSPACE/test/neoload/Frontend_neoload.yaml"
                     sh "sed -i 's/DTID_TO_REPLACE/${DYNATRACEID}/' $WORKSPACE/test/neoload/Frontend_neoload.yaml"
                     sh "sed -i 's/APIKEY_TO_REPLACE/${DYNATRACEAPIKEY}/'  $WORKSPACE/test/neoload/Frontend_neoload.yaml"
                     sh "sed -i 's,JSONFILE_TO_REPLACE,$WORKSPACE/monspec/catalogue_anomalieDection.json,'  $WORKSPACE/test/neoload/Frontend_neoload.yaml"
                     sh "sed -i 's/TAGS_TO_REPLACE/${NL_DT_TAG}/' $WORKSPACE/test/neoload/Frontend_neoload.yaml"
                     sh "sed -i 's,OUTPUTFILE_TO_REPLACE,$WORKSPACE/infrastructure/sanitycheck.json,'  $WORKSPACE/test/neoload/Frontend_neoload.yaml"

                      script {
                          neoloadRun executable: '/home/neoload/neoload/bin/NeoLoadCmd',
                                  project: "$WORKSPACE/test/neoload/load_template/load_template.nlp",
                                  testName: 'Stage_load_${VERSION}_${BUILD_NUMBER}',
                                  testDescription: 'Stage_load_${VERSION}_${BUILD_NUMBER}',
                                  commandLineOption: "-project  $WORKSPACE/test/neoload/Frontend_neoload.yaml -project $WORKSPACE/test/neoload/Frontend_neoload_sla.yaml -nlweb -L Population_Buyer=$WORKSPACE/infrastructure/infrastructure/neoload/lg/remote.txt -L Population_Dynatrace_Integration=$WORKSPACE/infrastructure/infrastructure/neoload/lg/local.txt -nlwebToken $NLAPIKEY -variables host=ec2-54-229-141-49.eu-west-1.compute.amazonaws.com,port=80",
                                  scenario: 'FrontEndLoad', sharedLicense: [server: 'NeoLoad Demo License', duration: 2, vuCount: 200],
                                  trendGraphs: [
                                          [name: 'Limit test Catalogue API Response time', curve: ['CatalogueList>Actions>Get Catalogue List'], statistic: 'average'],
                                          'ErrorRate'
                                  ]
                      }

                  }
                }

  /*  stage('DT Deploy Event') {
        when {
            expression {
            return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
            }
        }
        steps {
          container("curl") {
            // send custom deployment event to Dynatrace
            sh "curl -X POST \"$DT_TENANT_URL/api/v1/events?Api-Token=$DT_API_TOKEN\" -H \"accept: application/json\" -H \"Content-Type: application/json\" -d \"{ \\\"eventType\\\": \\\"CUSTOM_DEPLOYMENT\\\", \\\"attachRules\\\": { \\\"tagRule\\\" : [{ \\\"meTypes\\\" : [\\\"SERVICE\\\"], \\\"tags\\\" : [ { \\\"context\\\" : \\\"CONTEXTLESS\\\", \\\"key\\\" : \\\"app\\\", \\\"value\\\" : \\\"${env.APP_NAME}\\\" }, { \\\"context\\\" : \\\"CONTEXTLESS\\\", \\\"key\\\" : \\\"environment\\\", \\\"value\\\" : \\\"dev\\\" } ] }] }, \\\"deploymentName\\\":\\\"${env.JOB_NAME}\\\", \\\"deploymentVersion\\\":\\\"${_VERSION}\\\", \\\"deploymentProject\\\":\\\"\\\", \\\"ciBackLink\\\":\\\"${env.BUILD_URL}\\\", \\\"source\\\":\\\"Jenkins\\\", \\\"customProperties\\\": { \\\"Jenkins Build Number\\\": \\\"${env.BUILD_ID}\\\",  \\\"Git commit\\\": \\\"${env.GIT_COMMIT}\\\" } }\" "
          }
        }
    }*/
    /*stage('Mark artifact for staging namespace') {

      steps {

          sh "docker tag ${TAG_DEV} ${TAG_STAGING}"
          sh "docker push ${TAG_STAGING}"

      }
    }*/

    }
    post {
            always {
                    sh 'docker-compose -f $WORKSPACE/infrastructure/infrastructure/neoload/lg/docker-compose.yml down'
                    cleanWs()
                    sh 'docker volume prune'
            }

          }
  }

