#!/usr/bin/env groovy
node('build') {
    def image_ecr_name;
    def image_name;
    def scmVars;
    def mvn_version = 'Maven-Agent';
    // Jenkins pipeline parameters placeholders
    def pod_service_name = '';
    def pod_service_namespace = '';
    def pod_version = 'v1';
    def pod_replicas = 1;
    def pod_service_group = 'general';
    def pod_termination_grace_period_sec = 60;
    def service_enabled = false;
    def service_port = 8080;
    def host_entry='';
    def build_file;
    stage('Build Parameters  -> Setup') {
        def build_files = ["build-dev.xml", "build-sit.xml", "build-prod.xml", "build-uat.xml"];
        def jdk_versions = ["openjdk16", "jdk8"];
        def build_file_choices = buildChoices(build_files, "${params.ANT_BUILD_FILE}");
        def jdk_version_choices = buildChoices(jdk_versions, "${params.JDK_VERSION}")
        properties([
                parameters([
                        choice(name: "ANT_BUILD_FILE",
                                choices: build_file_choices.join("\n"),
                                description: 'This is used to specify ANT build file for external properties based on the environment you want to deploy service.'
                        ),
                        string(
                                name: 'NAMESPACE',
                                trim: true,
                                description: 'This is used to specify namespace that would hold pod and service. This namespace must already exists in the k8s cluster.',
                                defaultValue: params.NAMESPACE
                        ),
                        string(
                                name: 'NAME',
                                trim: true,
                                description: 'This is used to specify pod and service name.',
                                defaultValue: params.NAME
                        ),
                        string(
                                name: 'POD_VERSION',
                                trim: true,
                                description: 'This is used to specify pod version. e.g v1',
                                defaultValue: params.POD_VERSION ?: 'v1'
                        ),
                        string(
                                name: 'REPLICAS',
                                trim: true,
                                description: 'This is used to specify pod replicas. e.g 1',
                                defaultValue: params.REPLICAS ?: '1'
                        ),
                        string(
                                name: 'POD_TERMINATION_PERIOD_SECONDS',
                                trim: true,
                                description: 'This is used to specify pod termination grace period in seconds. Default value should be 60',
                                defaultValue: params.POD_TERMINATION_PERIOD_SECONDS ?: '60'
                        ),
                        string(
                                name: 'GROUP',
                                trim: true,
                                description: 'This is used to group pod/service resources in cluster',
                                defaultValue: params.GROUP ?: 'general'
                        ),
                        booleanParam(
                                description: 'This flag is used to create service for exposed pods',
                                name: 'CREATE_SERVICE',
                                defaultValue: params.CREATE_SERVICE ?: false
                        ),
                        string(
                                name: 'SERVICE_PORT',
                                trim: true,
                                description: 'This is used to specify port in case of exposed service, default value should be 8080',
                                defaultValue: params.SERVICE_PORT ?: '8080'
                        ),
                        text(
                                name: 'HOST_ENTRY',
                                trim: true,
                                description: 'This field is to add the host entry in host file, please take care of followings while editing this field \n' +
                                        "1. ip is a key for jenkins pipeline, dont change it \n" +
                                        "2. hostNames is also a key for jenkins pipeline, so dont change it as well\n" +
                                        "3. always keep hostNames values comma separated inside \"{ }\" \n" +
                                        "4. copy below sample template and edit it as per your need \n" +
                                        '[\n' +
                                        '   [ \n' +
                                        '       ip: "127.0.0.1", \n' +
                                        '       hostNames: "{localhost,local}"\n' +
										
                                       '   ] \n' +
//                                        '   [ \n' +
 //                                       '       ip: "192.168.1.2", \n' +
 //                                       '       hostNames: "{proxy.com,example.com}"\n' +
 //                                       '   ]\n' +
                                        ']',
                                defaultValue: params.HOST_ENTRY ?: ''
                        ),
                        choice(name: 'JDK_VERSION',
                                choices: jdk_version_choices.join("\n"),
                                description: 'This is used to specify JDK version to build your service.'
                        )
                ])
        ])
    }
    stage('Build Parameters -> Validate and Intialize') {
        if (!params.NAMESPACE?.trim() || !params.NAME?.trim() || !params.ANT_BUILD_FILE?.trim()) {
            error('Missing required parameters NAMESPACE/NAME/ANT_BUILD_FILE')
        } else {
            if (params.POD_VERSION?.trim()) { pod_version = params.POD_VERSION }
            if (params.REPLICAS?.trim()) { pod_replicas = params.REPLICAS }
            if (params.POD_TERMINATION_PERIOD_SECONDS?.trim()) { pod_termination_grace_period_sec = params.POD_TERMINATION_PERIOD_SECONDS }
            if (params.GROUP?.trim()) { pod_service_group = params.GROUP }
            if (params.SERVICE_PORT?.trim()) { service_port = params.SERVICE_PORT }
            if (params.HOST_ENTRY?.trim()) { host_entry = params.HOST_ENTRY }
            service_enabled = params.CREATE_SERVICE
            build_file = params.ANT_BUILD_FILE
            pod_service_name = params.NAME
            image_name = params.NAME
            pod_service_namespace = params.NAMESPACE
        }
    }
    stage('Clean workspace') {
        deleteDir()
        sh 'ls -lah'
    }
    stage('Git -> Checkout Source') {
        scmVars = checkout scm
    }
    stage('Copy -> Copy helm chart') {
        sh 'cp -r $chartPath .'
    }
    stage('Ant -> Build Properties') {
        // replace build files
        sh 'mv ' + build_file + ' build.xml'
        // Invoke ant
        withAnt(installation: 'default') {
            sh 'ant'
        }
    }
    stage('Maven -> Build Application') {
        echo 'Building application'
        if (params.JDK_VERSION != null && !params.JDK_VERSION.isEmpty()) {
            env.JAVA_HOME="${tool params.JDK_VERSION}"
        } else {
            env.JAVA_HOME="${tool 'jdk8'}"
        }
        withEnv( ["PATH+MAVEN=${tool mvn_version}/bin"] ) {
            sh 'mvn clean install -DskipTests=true'
        }
    }
    stage('ECR -> Create Registry') {
        timeout(10) {
            echo 'Creating registry if not already exist'
            sh '$ecr describe-repositories --repository-names ' + image_name  + ' || $ecr create-repository --repository-name ' +
                    image_name + ' --image-scanning-configuration scanOnPush=true --region $REGISTRY_REGION'
        }
    }
}
List buildChoices(List default_build_files, String previous_selection) {
    if (previous_selection == null) {
        return default_build_files;
    }
    def build_files = default_build_files.minus(previous_selection);
    build_files.add(0, previous_selection);
    return build_files;
}
String buildECRImageName(String image_name, String build_file) {
    def environment = build_file.substring(build_file.indexOf('-'));
    environment = environment.substring(0, environment.indexOf('.'));
    return image_name + environment;
}
String setHostAliases(def hostEntry) {
    if (hostEntry == null || "".equals(hostEntry.trim())) {
        return "--set hostAliases=null";
    }
    def hostAliases = "";
    def ent = evaluate(hostEntry);
    for (i=0; i < ent.size(); i++) {
        hostAliases=hostAliases + ' --set hostAliases[' + i + '].ip=' + ent.get(i).ip;
        hostAliases= hostAliases + ' --set "hostAliases[' + i + '].hostNames=' + ent.get(i).hostNames + '"'
    }
    return hostAliases;
}
