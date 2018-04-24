import groovy.io.FileType
import groovy.json.JsonSlurper
import groovy.json.JsonSlurperClassic
import groovy.json.JsonBuilder
import groovy.json.JsonOutput
import hudson.model.* 

node {
    stage('CheckoutPhase'){
      checkout scm
    } 

    stage('ValidateParametersPhase'){
	if (!"${ACTION}".toLowerCase().equals('new') || !"${ACTION}".toLowerCase().equals('update') || !"${ACTION}".toLowerCase().equals('delete')){
		println "Invalid Action parameter passed"
		       sh "exit 1"		
	} else if (("${ACTION}".toLowerCase().equals('new')) || ("${ACTION}".toLowerCase().equals('update'))){
		if ((!"${API_NAME}"?.trim()) || (!"${API_CTX}"?.trim()) || (!"${API_VERSION}"?.trim()) || (!"${WSDL_LOC}"?.trim()) || (!"${ENDPOINT}"?.trim()) || (!"${TARGET_ENV}"?.trim())){
			println "Mandatory parameter missing: Name= ${API_NAME} , Context= ${API_CTX} , Version= ${API_VERSION} , WSDL= ${WSDL_LOC} , Endpoint= ${ENDPOINT} , Environment= ${TARGET_ENV}"
		       sh "exit 1"
		}
		
	} else ("${ACTION}".toLowerCase().equals('delete')){
			println "Mandatory parameter missing: Name= ${API_NAME} , Context= ${API_CTX} , Version= ${API_VERSION}"
		       sh "exit 1"
	}
	
	
    }
 
 String context = "/"+"${API_CTX}"
 String versionWithEnv= "${TARGET_ENV}".toLowerCase() + "-" + "${API_VERSION}"
 String pathToApiMetadata= "${WORKSPACE}" + "/" + "${API_NAME}" + ".json"
 
 if(!"${ACTION}".toLowerCase().equals('delete')) {
        stage('CreateMetadataPhase'){     	
	pathToTemplate= "${WORKSPACE}" + "/ApiTemplate.json"
	
	def inputFile = new File(pathToTemplate) 
	def jsonSlurper = new JsonSlurper()
	def jsonObject = jsonSlurper.parse(inputFile)

	jsonObject.name = "${API_NAME}"
	jsonObject.context = context
	jsonObject.description = "${API_DESCRIPTION}"
	jsonObject.version = versionWithEnv
	jsonObject.wsdlUri= "${WSDL_LOC}"
	
	def endp = jsonSlurper.parseText(jsonObject.endpointConfig)
	endp.production_endpoints.url="${ENDPOINT}"
	String endpointString = JsonOutput.toJson(endp)
	jsonObject.endpointConfig = endpointString	
	new File(pathToApiMetadata).write(new JsonBuilder(jsonObject).toPrettyString())
    }
  }   
        stage('APIOperationPhase'){
	String api_action = "${ACTION}"
        def envProps = readJSON file: "${WORKSPACE}"+'/Env.json'
        String envPublish = envProps["${TARGET_ENV}".toLowerCase()]
        String cid = sh(script: "curl -k -X POST -H \"Authorization: Basic YWRtaW46YWRtaW4=\" -H \"Content-Type: application/json\" -d @payload.json https://${envPublish}:9443/client-registration/v0.11/register | jq -r \'.clientId\'",      returnStdout: true)
        String cs  = sh(script: "curl -k -X POST -H \"Authorization: Basic YWRtaW46YWRtaW4=\" -H \"Content-Type: application/json\" -d @payload.json https://${envPublish}:9443/client-registration/v0.11/register | jq -r \'.clientSecret\'", returnStdout: true)
        String cid_cs = cid.trim() + ":" + cs.trim()
        String encodeClient = cid_cs.bytes.encodeBase64().toString()
        String tokenCreate = sh(script:"curl -k -d \"grant_type=password&username=admin&password=admin&scope=apim:api_create\" -H \"Authorization: Basic ${encodeClient}\" https://${envPublish}:8243/token | jq -r \'.access_token\'", returnStdout: true)
	String tokenCreateTrimmed = tokenCreate.trim()

	if( api_action.toLowerCase().equals('new')) {	
		String createResponse = sh(script: "curl -k -H \"Authorization: Bearer ${tokenCreateTrimmed}\" -H \"Content-Type: application/json\" -X POST -d @${pathToApiMetadata} https://${envPublish}:9443/api/am/publisher/v0.11/apis", returnStdout: true)
                println createResponse
		def createResponseObj = readJSON text: createResponse
		String apiIDCreated = createResponseObj.id
		   if(apiIDCreated != null){
		       println "API created successfully."
		       apiIDCreated = apiIDCreated.trim()
		   } else {
		       println "API creation failed."
		       sh "exit 1"
		   }
		def tokenPublish = sh(script:"curl -k -d \"grant_type=password&username=admin&password=admin&scope=apim:api_publish\" -H \"Authorization: Basic ${encodeClient}\" https://${envPublish}:8243/token | jq -r \'.access_token\'", returnStdout: true)    
		sh "curl -k -H \"Authorization: Bearer ${tokenPublish}\" -X POST \"https://${envPublish}:9443/api/am/publisher/v0.11/apis/change-lifecycle?apiId=${apiIDCreated}&action=Publish\""	 
       }

       if( api_action.toLowerCase().equals('update')) {
    	      String tokenView = sh(script:"curl -k -d \"grant_type=password&username=admin&password=admin&scope=apim:api_view\" -H \"Authorization: Basic ${encodeClient}\" https://${envPublish}:8243/token | jq -r \'.access_token\'", returnStdout: true)
              def apisList = "["+sh(script:"curl -k -H \"Authorization: Bearer ${tokenView}\" https://${envPublish}:9443/api/am/publisher/v0.11/apis | jq \'.list\' | jq  \'.[] | {id: .id , name: .name , context: .context , version: .version}\'", returnStdout: true)+"]"
	      def formattedJson = apisList.replaceAll("\n","").replaceAll("\r", "").replaceAll("\\}\\{","\\},\\{")
	      def jsonProps = readJSON text: formattedJson
	      int count = 0
	      String updateId
              while(count < jsonProps.size()) {
                   def objAPI = readJSON text: jsonProps[count].toString()
                   if("${API_NAME}".toString().equals(objAPI.name) && context.equals(objAPI.context) && versionWithEnv.equals(objAPI.version)){
                     updateId =  objAPI.id
                     break
                   }
                   count++
              }
              if(updateId == null){
		       println "Update failed.API ID missing from WSO2."
		       sh "exit 1"
	      }
	     def json = new JsonSlurper().parse(new File(pathToApiMetadata))
	     json << ["id": updateId]
	     new File(pathToApiMetadata).write(JsonOutput.toJson(json))
	     println JsonOutput.toJson(json)
             json = null //Fix non serialazation exception.
	     def updateResponse = sh(script:"curl -k -H \"Authorization: Bearer ${tokenCreateTrimmed}\" -H \"Content-Type: application/json\" -X PUT -d @${pathToApiMetadata} https://${envPublish}:9443/api/am/publisher/v0.11/apis/${updateId}", returnStdout: true)
             println updateResponse
        }
      if( api_action.toLowerCase().equals('delete')) {
    	      String tokenView = sh(script:"curl -k -d \"grant_type=password&username=admin&password=admin&scope=apim:api_view\" -H \"Authorization: Basic ${encodeClient}\" https://${envPublish}:8243/token | jq -r \'.access_token\'", returnStdout: true)
              def apisList = "["+sh(script:"curl -k -H \"Authorization: Bearer ${tokenView}\" https://${envPublish}:9443/api/am/publisher/v0.11/apis | jq \'.list\' | jq  \'.[] | {id: .id , name: .name , context: .context , version: .version}\'", returnStdout: true)+"]"
	      def formattedJson = apisList.replaceAll("\n","").replaceAll("\r", "").replaceAll("\\}\\{","\\},\\{")
	      def jsonProps = readJSON text: formattedJson
	      int count = 0
	      String deleteId
              while(count < jsonProps.size()) {
                   def objAPI = readJSON text: jsonProps[count].toString()
                   if("${API_NAME}".toString().equals(objAPI.name) && context.equals(objAPI.context) && versionWithEnv.equals(objAPI.version)){
                     deleteId =  objAPI.id
                     break
                   }
                   count++
              }
              if(deleteId == null){
		       println "Delete failed.API ID missing from WSO2."
		       sh "exit 1"
	      }
            def deleteResponse = sh(script:"curl -k -H \"Authorization: Bearer ${tokenCreateTrimmed}\" -X DELETE https://${envPublish}:9443/api/am/publisher/v0.11/apis/${deleteId}", returnStdout: true)
        }

    }
}
