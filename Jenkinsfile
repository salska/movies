String envByTagName() {
	 String tagName =  "${env.TAG_NAME}";
	 
	 String[] str = tagName.split('-');
	 for (String values: str) {
		 println(values);
		 
		 if (str[0] == 'dev' || str[0] == 'sit' || str[0] == 'uat' || str[0] == 'pte'|| str[0] == 'cpte'|| str[0] == 'prod') {
			return str[0].toString();
		 }
	 }
	 return 'dev';
}

void sendMail(String SUBJECT,String RECEIPIENTS,String MESSAGE){

	/*emailext body: "$MESSAGE",
			subject: "${SUBJECT}",
			to: "${RECEIPIENTS}";*/

	sh "mailx -s \"${SUBJECT}\" ${RECEIPIENTS} <<< \"${MESSAGE}\" ";

}

void sendMailAttach(String SUBJECT,String RECEIPIENTS, String MESSAGE,String ATTACHMENTS){

    /*emailext body: "$MESSAGE",
			subject: "${SUBJECT}",
			to: "${RECEIPIENTS}",
			attachmentsPattern: "${ATTACHMENTS}",
			attachLog: true; */

	sh "mailx -s \"${SUBJECT}\" -a \"${ATTACHMENTS}\" ${RECEIPIENTS} <<< \"${MESSAGE}\" ";

}

void sendNotify(){
	String body = "${currentBuild.currentResult}: Job ${env.JOB_NAME} build ${env.BUILD_NUMBER}\n More info at: ${env.BUILD_URL}";
	String subject= "Jenkins Build ${currentBuild.currentResult}: Job ${env.JOB_NAME}";
	String to= "${params.TOMAIL}";
	//sending mail after completion
	try {
		sendMail(subject,to,body);
	} catch (error) {
		echo "Mail Notify failed";
	}
}
void replaceByENV(String fileName){
	replaceInYAML("image","       image: ${env.ECRIMAGENAME}",fileName);
	replaceInYAML("replicas"," replicas: ${params.REPLICAS}",fileName);
}
void replaceInProp(String searchword,String newWord, String filename){
	sh "sed -i '/${searchword}/c\\${newWord}' ${filename} ";
}
void replaceInENV(String fileName,String pass){
	replaceInProp("TF_VAR_rdsPass","TF_VAR_rdsPass=${pass}",fileName);
}
void sendTestReportMail() {
	String body =  "Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n Please find attached Jacoco Report. \n More info at: ${env.BUILD_URL}";
	String subject =  "Jenkins Build Job ${env.JOB_NAME} - Jacoco Test Report";
	String to =  "${params.TOMAIL}";
	//sending mail after completion
	try {
		sendMailAttach(subject,to,body,'jreport.zip');
	} catch (error) {
		echo "Jacoco Report mail notify failed";
	}

}

void sendSonarReportMail() {
	String body =  "Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n Please find attached Sonar Report. \n More info at: ${env.BUILD_URL}";
	String subject =  "Jenkins Build Job ${env.JOB_NAME} - Sonar Report";
	String to =  "${params.TOMAIL}";
	//sending mail after completion
	try {
		sendMailAttach(subject,to,body,'sonar.zip');
	} catch (error) {
		echo "Sonar Report mail notify failed";
	}

}
def healthCheck() {
	def url= "${params.ENDPOINT}";
	def response = httpRequest url;
	echo "URL: "+ url +"---Status: "+ response.status;
	return response;
}

pipelineOLD {
	agent {
		label 'docker-slave-cluster'
	}
	
	environment {
		BBCREDENTIALS = credentials('BBCREDENTIALS');
		JENKINS_CREDENTIALS = credentials('jenkadm');
	}

	// Set log rotation, timeout and timestamps in the console
	options {
		disableConcurrentBuilds()
		buildDiscarder(logRotator(numToKeepStr: '5'));
		timestamps();
		timeout(time: 10, unit: 'MINUTES');
		//copyArtifactPermission(env.JOB_NAME);
        copyArtifactPermission('*')
	}
	
	parameters {
        choice( name: 'DEPLOY_ENV', choices: ['DEV','SIT','UAT','PRE','PROD'], description: 'Environment to Deploy from Branch!');
        booleanParam(name: 'SONAR_USE', defaultValue: true, description: 'Sonar Qube Check!');
        booleanParam(name: 'QUALITY_GATE', defaultValue: false, description: 'Quality Gate Check?');
        //string(name: 'TOMAIL', defaultValue: 'devops-support@morrisonsplc.co.uk', description: 'Default Recipient');
        string(name: 'TOMAIL', defaultValue: 'salih.kabay@morrisonsplc.co.uk', description: 'Default Recipient');
        string(name: 'RDS_PASS', defaultValue: 'xxxx', description: 'RDS Pass?');
    }
	
	stages {
        
		stage('Tools: Check') {
            steps {
                script {

                    //add environment name
                    def ENVNAME="";
                    if(params.TAG_DEPLOY){
                        ENVNAME= envByTagName();
                    }else{
                        ENVNAME= "${params.DEPLOY_ENV}";
                    }
                    env.ENVNAME = "${ENVNAME.toLowerCase()}";

                    //Get commit hash
                    def GIT_COMMIT_HASH="-";
                    try{
                        GIT_COMMIT_HASH += sh (script: "git log -n 1 --pretty=format:'%H'", returnStdout: true);
                    }catch (error) {
                        GIT_COMMIT_HASH = "";
                    }

                    //assign ECR tag and image name
                    env.ECRTAG="${env.ENVNAME}"
                    env.ECRIMAGENAME="${params.ECRREPO}:${env.ECRTAG}";
                }
            }
        }
		
		stage('Build') {
            steps {
                script {
                    //input message: "Proceed to ${params.DEPLOY_ENV} deployment? (Click 'Proceed' to continue)";
                    echo "Build Started";

                    sh 'mvn -s mvn-settings.xml clean -Pdev';
                    sh 'mvn -s mvn-settings.xml install -f pom.xml -Dmaven.test.skip=true -Pdev';
                    sh 'cat ./commercial-hbtw-camunda-range-change/target/classes/application.properties';
                    def configENVFile="commercial-hbtw-camunda-range-change/src/main/resources/application.properties";
                    env.configENVFile=configENVFile;

                    sh "zip -r app.zip ./commercial-hbtw-camunda-range-change/target/*.jar Dockerfile ./commercial-hbtw-camunda-range-change/ops/runApp ./${env.configENVFile}";
                    archiveArtifacts artifacts: 'app.zip', excludes: null, fingerprint: true, onlyIfSuccessful: true;
                    echo "Build Completed";
                }
            }
        }

        stage('Test: Coverage') {
            options {
                skipDefaultCheckout true;
            }
            when {
                expression { return params.SONAR_USE;	}
            }
            steps {
                script {
                    echo "Coverage Test: Sonar";
                    try {
                        withSonarQubeEnv('sonarqube') {
                            sh 'mvn sonar:sonar -Dsonar.login=3a155869bdf0432584bb378bedaf1a5ffe80e97a'
                        }

                        sh 'zip -r sonar.zip ./commercial-hbtw-camunda-range-change/target/sonar ';
                        archiveArtifacts artifacts: 'sonar.zip', excludes: null, fingerprint: true, onlyIfSuccessful: true;

                        if(params.QUALITY_GATE){
                            timeout(time: 5, unit: 'MINUTES') {
                                def qg = waitForQualityGate()
                                  if (qg.status != 'OK') {
                                      error "Pipeline aborted due to quality gate failure: ${qg.status}"
                                  }
                             }
                        }

                    } catch (err) {
                        echo "Sonar failed";
                    }
                    echo "Coverage Test: Sonar Completed";
                }
            }
        }

        stage('Create infrastructure and deploy code to dev') {

            environment {
                API_KEY = 'true'
                API_SECRET = 'sqlite'
            }
            agent {
                node { label 'docker-slave-cluster' }
            }

            when { branch 'develop' }
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'commercial-aws-admin']]) {
                    script {
						replaceInENV("commercial-hbtw-camunda-range-change/configs/develop/dev/application.properties","${params.RDS_PASS}");
                        dir("commercial-hbtw-camunda-range-change/ops/terraform/ecs-repo") {
                            sh ". ../../../utils.sh && execute runTerraform"
                        }
                        sh 'mvn -s mvn-settings.xml clean -Pdev';
                        sh 'mvn -s mvn-settings.xml install -f pom.xml -Dmaven.test.skip=true -Pdev';
                        sh '`aws ecr get-login --no-include-email --region eu-west-1`'
                       echo 'Docker DEV Infra Build..'
                      // copy into directory
                      dir('commercial-hbtw-camunda-range-change'){
                           sh 'mkdir -p docker_build_01'
                           sh 'cp -f target/camrra.jar docker_build_01/camrra.jar'

                           sh 'cp -rf ops/docker/* docker_build_01/'
                           sh 'cp -f ops/runApp docker_build_01/'
                           sh 'chmod 777 -R docker_build_01/*'
                           dir('docker_build_01'){
                                sh '. ../utils.sh && execute buildTagAndPushContainer dev Dockerfile'
                           }
                       }

                       dir("commercial-hbtw-camunda-range-change/ops/terraform/ecs-service") {
                            sh '. ../../../utils.sh && execute runTerraform'
                        }
                        echo 'Docker DEV Infra Build .. Completed'
                    }
                }
            }
        }

        stage('Create infrastructure and deploy code to sit') {

            environment {
                API_KEY = 'true'
                API_SECRET = 'sqlite'
            }
            when {
                //expression { return params.DEPLOY_ENV == 'SIT';	}
                tag 'sit-*'
            }
            agent {
                node { label 'docker-slave-cluster' }
            }
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'commercial-aws-admin']]) {
                    script {
                        replaceInENV("commercial-hbtw-camunda-range-change/configs/release/sit/application.properties","${params.RDS_PASS}");
                        sh 'mvn -s mvn-settings.xml clean -Psit';
                        sh 'mvn -s mvn-settings.xml install -f pom.xml -Dmaven.test.skip=true -Psit';
                        sh '`aws ecr get-login --no-include-email --region eu-west-1`'
                       echo 'Docker SIT Infra Build..'
                      // copy into directory
                      dir('commercial-hbtw-camunda-range-change'){
                           sh 'mkdir -p docker_build_01'
                           sh 'cp -f target/camrra.jar docker_build_01/camrra.jar'

                           sh 'cp -rf ops/docker/* docker_build_01/'
                           sh 'cp -f ops/runApp docker_build_01/'
                           sh 'chmod 777 -R docker_build_01/*'
                           dir('docker_build_01'){
                                sh '. ../utils.sh && execute buildTagAndPushContainer sit Dockerfile'
                           }
                       }

                       dir("commercial-hbtw-camunda-range-change/ops/terraform/ecs-service") {
                            sh '. ../../../utils.sh && execute runTerraform'
                        }
                        echo 'Docker SIT Infra Build .. Completed'
                    }
                }
            }
        }

        stage('Create infrastructure and deploy code to uat') {

                    environment {
                        API_KEY = 'true'
                        API_SECRET = 'sqlite'
                    }
                    when {
                        //expression { return params.DEPLOY_ENV == 'UAT';	}
                        tag 'uat-*'
                    }
                    agent {
                        node { label 'docker-slave-cluster' }
                    }
                    steps {
                        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'commercial-aws-admin']]) {
                            script {
                                replaceInENV("commercial-hbtw-camunda-range-change/configs/release/uat/application.properties","${params.RDS_PASS}");
                                sh 'mvn -s mvn-settings.xml clean -Puat';
                                sh 'mvn -s mvn-settings.xml install -f pom.xml -Dmaven.test.skip=true -Puat';
                                sh '`aws ecr get-login --no-include-email --region eu-west-1`'
                               echo 'Docker UAT Infra Build..'
                              // copy into directory
                              dir('commercial-hbtw-camunda-range-change'){
                                   sh 'mkdir -p docker_build_01'
                                   sh 'cp -f target/camrra.jar docker_build_01/camrra.jar'

                                   sh 'cp -rf ops/docker/* docker_build_01/'
                                   sh 'cp -f ops/runApp docker_build_01/'
                                   sh 'chmod 777 -R docker_build_01/*'
                                   dir('docker_build_01'){
                                        sh '. ../utils.sh && execute buildTagAndPushContainer uat Dockerfile'
                                   }
                               }

                               dir("commercial-hbtw-camunda-range-change/ops/terraform/ecs-service") {
                                    sh '. ../../../utils.sh && execute runTerraform'
                                }
                                echo 'Docker UAT Infra Build .. Completed'
                            }
                        }
                    }
                }

        stage('Create infrastructure and deploy code to pre') {

            environment {
                API_KEY = 'true'
                API_SECRET = 'sqlite'
            }
            when {
                //expression { return params.DEPLOY_ENV == 'PRE';	}
                tag 'pre-*'
            }
            agent {
                node { label 'docker-slave-cluster' }
            }
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'commercial-aws-admin']]) {
                    script {
                        replaceInENV("commercial-hbtw-camunda-range-change/configs/release/pre/application.properties","${params.RDS_PASS}");
                        sh 'mvn -s mvn-settings.xml clean -Ppre';
                        sh 'mvn -s mvn-settings.xml install -f pom.xml -Dmaven.test.skip=true -Ppre';
                        sh '`aws ecr get-login --no-include-email --region eu-west-1`'
                       echo 'Docker PRE Infra Build..'
                      // copy into directory
                      dir('commercial-hbtw-camunda-range-change'){
                           sh 'mkdir -p docker_build_01'
                           sh 'cp -f target/camrra.jar docker_build_01/camrra.jar'

                           sh 'cp -rf ops/docker/* docker_build_01/'
                           sh 'cp -f ops/runApp docker_build_01/'
                           sh 'chmod 777 -R docker_build_01/*'
                           dir('docker_build_01'){
                                sh '. ../utils.sh && execute buildTagAndPushContainer pre Dockerfile'
                           }
                       }

                       dir("commercial-hbtw-camunda-range-change/ops/terraform/ecs-service") {
                            sh '. ../../../utils.sh && execute runTerraform'
                        }
                        echo 'Docker PRE Infra Build .. Completed'
                    }
                }
            }
        }
        stage('Docker pull pre image') {

            environment {
                API_KEY = 'true'
                API_SECRET = 'sqlite'
            }
            when {
                //expression { return params.DEPLOY_ENV == 'PRE';	}
                tag 'pre-*'
            }
            agent {
                node { label 'tedoc04x' }
            }
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'commercial-aws-admin']]) {
                    script {
                        sh 'eval $(aws ecr get-login --region eu-west-1 --no-include-email)';
                        sh 'docker pull 995766112201.dkr.ecr.eu-west-1.amazonaws.com/commercial/camrrav3:pre'
                    }
                }
            }
        }
    }
	post {
		always {
			
			sendNotify(); /* Send Mail Notification */
			  sendTestReportMail();
			  sendSonarReportMail();
			  deleteDir() /* clean up our workspace */
			  cleanWs(); /* Clean up our workspace */
			
		}
	}
}
pipeline {
    agent any

    stages {
        stage('Init') {
            steps {
                echo 'Initializing..'
                echo "Running ${env.BUILD_ID} on ${env.JENKINS_URL}"
            }
        }
        stage('Test') {
            steps {
                echo 'Testing..'
                echo 'Running pytest..'
            }
        }
        stage('Build') {
            steps {
                echo 'Building..'
                echo 'Running docker build -t sntshk/cotu .'
            }
        }
        stage('Publish') {
            steps {
                echo 'Publishing..'
                echo 'Running docker push..'
            }
        }
        stage('Cleanup') {
            steps {
                echo 'Cleaning..'
                echo 'Running docker rmi..'
            }
        }
    }
}
