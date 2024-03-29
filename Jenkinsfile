pipeline {
	
  agent any
	
  environment {
	  VENV_DIR = 'OS10_CICD_venv'
	  PYVERSION = 'python3'
	  PYBINPATH = 'bin'
  }
	
  stages {
	  
	stage('Build') {
		steps {
			sh '${PYVERSION} -m venv .${VENV_DIR}'
			sh '. .${VENV_DIR}/${PYBINPATH}/activate'
			sh '.${VENV_DIR}/${PYBINPATH}/${PYVERSION} -m pip install -r pyrequirements.txt'
			sh '.${VENV_DIR}/${PYBINPATH}/${PYVERSION} -m py_compile startcicd.py'
			stash(name: 'compiled-results', includes: '*.py*')
		}
	}

    	stage('Show host versions') {
      		steps {
			echo 'Show ${PYVERSION} versions:'
        		sh '.${VENV_DIR}/${PYBINPATH}/${PYVERSION} --version'
			sh '.${VENV_DIR}/${PYBINPATH}/${PYVERSION} -m pip list'
      		}
    	}
	//This stage is to spare on resources in the Compute platform (Dev & Prod run together gives problems) 
    	/*stage('Stop GNS3 Stage PROD') {
      		steps {
			echo 'Shutting down Prod fabric to conserve compute resources..'
        		sh 'python3 -u startcicd.py stopgns3 prodstage'
			sleep( time: 3 )
      		}
	}*/
	stage('Stage Dev: Provision GNS3 Dev network.....') {
		
		environment {
			LS = "${sh(script:'.${VENV_DIR}/${PYBINPATH}/${PYVERSION} -u startcicd.py creategns3project devstage | grep "proceed"', returnStdout: true).trim()}"
    		}
      		
		steps {
			script {
				println "Feedback from python script: ${env.LS}"
				if (env.LS == 'proceed = True') {
					env.noztpcheck = ''
					echo 'Dev Network provisioning finished. Proceed to Stage Dev: Start Dev network.'
					echo 'This can take ~45 minutes if ZTP staging is involved.....'
                                        sleep( time: 2 )
                                }
				else if  (env.LS == 'proceed = noztp_check') {
					env.noztpcheck = 'noztp_check'
					echo 'Project already exists in GNS3. Nodes will start without ZTP.'
					echo 'Wait for nodes to become ready while booting...'
					sleep( time: 2 )
				}
				else {
					echo 'Job execution to provision Dev stage failed.'
					println "${env.LS}, EXIT with errors."
            				error ("There were failures in the job template execution. Pipeline stops here.")
                                }
			}
      		}
	}

    	stage('Stage Dev: Start GNS3 ZTP staging.....') {

		environment {
			LS = "${sh(script:'.${VENV_DIR}/${PYBINPATH}/${PYVERSION} -u startcicd.py startgns3 devstage ${noztpcheck} | grep "proceed"', returnStdout: true).trim()}"
		}
		
		steps {
			script {
				//echo "${env.LS}"
				//echo "${noztpcheck}"
				println "Feedback from python script: ${env.LS}"
				if (env.LS == 'proceed = True') {
					echo 'Dev network succesfully started. Proceed to Stage Dev: Configure Dev network.'
					echo 'This can take ~25 minutes.....'
                                        sleep( time: 10 )
                                }
				else {
					echo 'Job execution to start Dev stage failed.'
					println "${env.LS}, EXIT with errors."
            				error ("There were failures while waiting to dev network becomes ready to configure. Pipeline stops here.")
                                }
			}
      		}
	}

	stage("Stage Dev: Configure GNS3 Dev network....") {

		environment {
			LS = "${sh(script:'.${VENV_DIR}/${PYBINPATH}/${PYVERSION} -u startcicd.py launchawx devstage configure | grep "proceed"', returnStdout: true).trim()}"
    		}
                            
		steps {			
			script {
				println "Feedback from python script: ${env.LS}"
				echo 'Configuration Job finished.'
				//echo "${env.LS}"
				if (env.LS == 'proceed = True') { //100% oke
					sleep( time: 10 )
					echo 'Succesfull Job completion.'
            				echo 'Proceed to Stage Dev fase Validate operational status.'
					echo 'This can take some minutes...'
				}
				if (env.LS.indexOf('relaunch') != -1) { //a relaunch was proposed, there were failures
					relaunchuri = env.LS.substring(env.LS.lastIndexOf('=') + 1, env.LS.length())
					//println "${relaunchuri}"
					echo 'There are failures in Ansible playbook run. Retrying once on failed hosts...'
					sleep( time: 2 )
					env.RL = "${sh(script:""".${VENV_DIR}/${PYBINPATH}/${PYVERSION} -u startcicd.py launchawx relaunch $relaunchuri | grep 'proceed'""", returnStdout: true).trim()}"
					//echo "${env.RL}" //Show for logging, clearity
					println "Feedback from python script: ${RL}"
					if (env.RL == 'proceed = True') { //100% oke
						echo 'Succesfull Job completion.'
            					echo 'Proceed to Stage Dev fase Validate operational status.'
						echo 'This can take some minutes...'
						sleep( time: 5 )
					} else {
						println "Concurrent failures in Configuration Job. ${env.RL}, EXIT with error from else statement."
						error ("There are concurrent failures in the job template execution. Pipeline stops here.")
					}
        			}
				if (env.LS == 'proceed = False') {
					echo 'Job execution failed.'
					println "${env.LS}, EXIT with error from last if (env.LS) statement."
            				error ("There were failures in the job template execution. Pipeline stops here.")
        			}
			}
		}
        }
	stage("Stage Dev: GNS3 Run Closed loop Validation tests") {
		environment {
			LS = "${sh(script:'.${VENV_DIR}/${PYBINPATH}/${PYVERSION} -u startcicd.py launchawx devstage test | grep "proceed"', returnStdout: true).trim()}"
    		}
            
		steps {
			script {
				println "Feedback from python script: ${env.LS}"
				echo 'Closed loop Validation succesfully finished.'
				//echo "${env.LS}"
				if (env.LS == 'proceed = True') {
					echo 'All validation tests succesful.'
					sleep( time: 2 )
					//This step is to spare on resources in the Compute platform (Dev & Prod run together gives problems) 
					//echo 'Will decommision Dev network to spare GNS3 resources...'
					//sleep( time: 2 )
					//sh '.${VENV_DIR}/${PYBINPATH}/${PYVERSION} -u startcicd.py stopgns3 devstage' //Stop GNS3 project
            				//echo 'Proceed to Stage Prod fase Provisioning.'
					sleep( time: 3 )
        			} else {
					echo 'Closed loop Validation tests failed! Feeback from script:'
					echo "${env.LS}"
            				error ("There were failures in the job template execution. Pipeline stops here.")
        			}
			}
		}
        }
	  
	  
    	/*
	stage('Stage Prod: Provision GNS3 prod network') {
		
		environment {
			LS = "${sh(script:'python3 -u startcicd.py startgns3 prodstage | grep "proceed"', returnStdout: true).trim()}"
    		}
      		
		steps {
			script {
				//echo "${env.LS}" 
				if (env.LS == 'proceed = True') {
					echo 'Prod network is already provisioned.'
					echo 'Will proceed to Stage Prod: Configure network.'
					echo 'This can take ~15 mins...'
                                        sleep( time: 2 )
                                }
				else {
					//GNS3 API call to start Network has just been done by startcicd.py script
					echo 'Prod network is being provisioned. This can take ~3 mins.'
					echo 'Waiting for systems te become active...'
					sleep( time: 180 )
					echo 'Done. Systems active.'
                                }
			}
      		}
	}

	stage("Stage Prod: Configure Prod network") {
		environment {
			LS = "${sh(script:'python3 -u startcicd.py launchawx prodstage configure | grep "proceed"', returnStdout: true).trim()}"
			relaunchuri = ""
    		}
                            
		steps {
			script {
				echo 'Configuration of Prod network finished.'
				echo 'Waiting 10 secs to let the network converge...'
				//echo "${env.LS}"
				if (env.LS == 'proceed = True') { //100% oke
					sleep( time: 10 )
					echo 'Succesfull Job completion.'
            				echo 'Proceed to Stage Prod fase Ping Tests.'
					echo 'This can take some minutes...'
				}
				if (env.LS.indexOf('relaunch') != -1) { //a relaunch was proposed, there were failures
					relaunchuri = env.LS.substring(env.LS.lastIndexOf('=') + 1, env.LS.length())
					println "${relaunchuri}"
					echo 'There are failures in ansible playbook run. Retrying once on failed hosts...'
					sleep( time: 2 )
					env.RL = "${sh(script:"""python3 -u startcicd.py launchawx relaunch $relaunchuri | grep 'proceed'""", returnStdout: true).trim()}"
					//echo "${env.RL}" //Show for logging, clearity
					
					if (env.RL == 'proceed = True') { //100% oke
						echo 'Succesfull Job completion.'
            					echo 'Proceed to Stage Prod fase Ping Tests.'
						echo 'This can take some minutes...'
						sleep( time: 5 )
					} else {
						println "Concurrent failures in Configuration Job. ${env.RL}, EXIT with error from else statement."
						error ("There are concurrent failures in the job template execution. Pipeline stops here.")
					}
        			}
				if (env.LS == 'proceed = False') {
					echo 'Job execution failed.'
					println "${env.LS}, EXIT with error from last if (env.LS) statement."
            				error ("There were failures in the job template execution. Pipeline stops here.")
        			}
			}
		}
        }

	stage("Stage Prod: Run connectivity Tests") {
		environment {
			LS = "${sh(script:'python3 -u startcicd.py launchawx prodstage test | grep "proceed"', returnStdout: true).trim()}"
    		}
            
		steps {
			script {
				echo 'Job pingtests has Finished on Prod network.'
				//echo "${env.LS}"
				if (env.LS == 'proceed = True') {
					echo 'All pingtests succeeded.'
					sleep( time: 2 )
            				echo 'The production network runs fine with the new changes. Well done! :-)'
					sleep( time: 2 )
        			} else {
					echo 'ALERT - ALERT - ALERT.'
					echo 'Production network could be down or affect connectivity currently.'
					echo 'Pingtests failed! Something wrong in the change code?'
					echo 'Advise: rollback your change.'
            				error ("There were failures in the job template execution. Pipeline stops here.")
        			}
			}
		}
        }
	*/

  }
}

