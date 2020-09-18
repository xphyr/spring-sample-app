// path of the template to use
def templatePath = 'veermuchandi/spring-mvn-base~https://github.com/xphyr/spring-sample-app'
// name of the template that will be created
def templateName = 'spring-sample-app'
// NOTE, the "pipeline" directive/closure from the declarative pipeline syntax needs to include, or be nested outside,
// and "openshift" directive/closure from the OpenShift Client Plugin for Jenkins.  Otherwise, the declarative pipeline engine
// will not be fully engaged.
pipeline {
    agent {
        node {
        // spin up a node.js slave pod to run this build on
        label 'maven'
        }
    }
    options {
        // set a timeout of 20 minutes for this pipeline
        timeout(time: 20, unit: 'MINUTES')
    }

    stages {
        stage('preamble') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject('development') {
                            echo "Using project: ${openshift.project()}"
                        }
                    }
                }
            }
        }
        stage('cleanup') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject('development') {
                            // delete everything with this template label
                            openshift.selector("all", [ template : templateName ]).delete()
                            // delete any secrets with this template label
                            if (openshift.selector("secrets", templateName).exists()) {
                                openshift.selector("secrets", templateName).delete()
                            }
                        }
                    }
                } // script
            } // steps
        } // stage

        stage('create') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject('development') {
                            if (openshift.selector("bc", templateName).exists()) {
                                openshift.selector("bc", templateName).startBuild();
                            }
                            else {
                                // create a new application from the templatePath
                                openshift.newApp(templatePath).narrow('svc').expose();
                            }
                        }
                    }
                } // script
            } // steps
        } // stage
        stage('build') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject('development') {
                            def builds = openshift.selector("bc", templateName).related('builds')
                            builds.untilEach(1) {
                                return (it.object().status.phase == "Complete")
                            }
                        }
                    }
                } // script
            } // steps
        } // stage
        stage('deploy') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject('development') {
                            // def rm = openshift.selector("deploy", templateName).rollout()
                            openshift.selector("pod", [deployment : "${templateName}"]).untilEach(1) {
                                return (it.object().status.phase == "Running")
                            }
                        }
                    }
                } // script
            } // steps
        } // stage
        ///* - we will remove this later
        stage('Tag for Staging') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject('development') {
                            openshift.tag("${templateName}:latest", "${templateName}-staging:latest")
                        }
                    }
                } // script
            } // steps
        } // stage
        stage ('Create Testing Deployment') {
            when {
                expression {
                    openshift.withCluster() {
                        openshift.withProject('testing') {
                            return !openshift.selector("pod", [deployment : "${templateName}-staging"]).exists()
                        }
                    }
                }
            }
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject('testing') {
                            openshift.newApp("image-registry.openshift-image-registry.svc:5000/development/${templateName}-staging:latest").narrow('svc').expose()
                        }
                    }
                }
            }
        }
        stage('Validate Staging') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject('testing') {
                            // def rm = openshift.selector("deploy", templateName).rollout()
                            openshift.selector("pod", [deployment : "${templateName}-staging"]).untilEach(1) {
                                return (it.object().status.phase == "Running")
                            }
                        }
                    }
                } // script
            } // steps
        } // stage
        stage('Promote to Production') {
            steps {
                input "Shal we promote to production"
            }
        }
        stage('Tag for Promotion') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject('development') {
                            openshift.tag("${templateName}:latest", "${templateName}-production:latest")
                        }
                    }
                } // script
            } // steps
        } // stage
        stage ('Create Production Deployment') {
            when {
                expression {
                    openshift.withCluster() {
                        openshift.withProject('production') {
                            return !openshift.selector("pod", [deployment : "${templateName}-production"]).exists()
                        }
                    }
                }
            }
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject('production') {
                            openshift.newApp("image-registry.openshift-image-registry.svc:5000/development/${templateName}-production:latest").narrow('svc').expose()
                        }
                    }
                }
            }
        }
        stage('Validate Production') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject('production') {
                            // def rm = openshift.selector("deploy", templateName).rollout()
                            openshift.selector("pod", [deployment : "${templateName}-production"]).untilEach(1) {
                                return (it.object().status.phase == "Running")
                            }
                        }
                    }
                } // script
            } // steps
        } // stage
        // we will remove this line later */
    } // stages
} // pipeline