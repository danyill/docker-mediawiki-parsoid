#!/usr/bin/env groovy
registry_url = "https://index.docker.io/v1/"
docker_creds_id = "rlewkowicz"
build_tag = "testing"
maintainer_name = "rlewkowicz"
container_name = "php-fpm"
node{
  retry( count: 3  ){
    timeout(time: 5, unit: 'SECONDS') {
      stage('Pull and Update') {
        git url: 'https://github.com/rlewkowicz/php.git'
        sh './update.sh'
      }
    }
  }
  stage('Build') {
    echo "Building PHP-FPM with docker.build(${maintainer_name}/${container_name}:${build_tag})"
    container = docker.build("${maintainer_name}/${container_name}:${build_tag}", '7.0/fpm/alpine')
  }
  stage('Test') {
    try {

      docker.image("${maintainer_name}/${container_name}:${build_tag}").withRun("--name=${container_name} -d -p 127.0.0.1:8000:8000", "php -S 0.0.0.0:8000 -t / /phpinfo.php" )  { c ->

         waitUntil {
             sh "docker exec -t ${container_name} netstat -apn | grep 8000 | grep LISTEN | wc -l | tr -d '\n' > /tmp/wait_results"
             wait_results = readFile '/tmp/wait_results'

             echo "Wait Results(${wait_results})"
             if ("${wait_results}" == "1")
             {
                 echo "PHP is listening on port 8000"
                 sh "rm -f /tmp/wait_results"
                 return true
             }
             else
             {
                 echo "PHP is not listening yet"
                 return false
             }
         }

         echo "PHP is running"

         MAX_TESTS = 2
         for (test_num = 0; test_num < MAX_TESTS; test_num++) {

             echo "Running Test(${test_num})"

             expected_results = 0
             if (test_num == 0 )
             {
                 test_results = sh([script: "curl -s 127.0.0.1:8000 | grep -o 'PHP Version 7.0'", returnStatus:true])
                 build_tag = sh([script: $/curl -s 127.0.0.1:8000 | grep -o "PHP Version 7.0.[0-9]*" | grep -o 7.0.[0-9]*/$, returnStdout: true])
                 expected_results = 0
             }
             else if (test_num == 1)
             {
                 // Test that port 80 is exposed
                 echo "Exposed Docker Ports:"
                 test_results = sh([script: "docker inspect --format '{{ (.NetworkSettings.Ports) }}' ${container_name} | grep map | grep '9000/tcp:'", returnStatus:true])
                 expected_results = 0
             }
             else
             {
                 err_msg = "Missing Test(${test_num})"
                 echo "ERROR: ${err_msg}"
                 currentBuild.result = 'FAILURE'
                 error "Failed to finish container testing with Message(${err_msg})"
             }

             // Now validate the results match the expected results
             stage ("Test(${test_num}) - Validate Results"){
               echo "Done Running Test(${test_num})"
               if (test_results != expected_results){ currentBuild.result = 'FAILURE' }
             }
         }
      }

      } catch (Exception err) {
        err_msg = "Test had Exception(${err})"
        currentBuild.result = 'FAILURE'
        error "FAILED - Stopping build for Error(${err_msg})"
      }
  }
  stage('Tag & Upload') {
    echo "Current Build ${build_tag}"
    container.tag([build_tag])
    container.push([build_tag])
    echo "Pushed Build ${build_tag}"
  }
}
