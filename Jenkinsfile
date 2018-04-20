import groovy.io.FileType
import groovy.json.JsonSlurper
import groovy.json.JsonBuilder
import groovy.json.JsonOutput

node {
    stage('CheckoutPhase'){
      checkout scm
    } 
 
 def context = "/"+"${API_NAME}"
 def versionWithEnv= "${TARGET_ENV}".toLowerCase() + "-" + "${API_VERSION}"
 
 if(!"${ACTION}".toLowerCase().equals('delete')) {
        stage('CreateMetadataPhase'){     	
	pathToTemplate= "${WORKSPACE}" + "/ApiTemplate.json"
	pathToApiMetadata= "${WORKSPACE}" + "/" + "${API_NAME}" + ".json"
	
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
	def api_action = "${ACTION}"
        def envProps = readJSON file: "${WORKSPACE}"+'/Env.json'
        def envPublish = envProps["${TARGET_ENV}".toLowerCase()]
        def cid = sh(script: "curl -k -X POST -H \"Authorization: Basic YWRtaW46YWRtaW4=\" -H \"Content-Type: application/json\" -d @payload.json https://${envPublish}:9443/client-registration/v0.11/register | jq -r \'.clientId\'",      returnStdout: true)
        def cs  = sh(script: "curl -k -X POST -H \"Authorization: Basic YWRtaW46YWRtaW4=\" -H \"Content-Type: application/json\" -d @payload.json https://${envPublish}:9443/client-registration/v0.11/register | jq -r \'.clientSecret\'", returnStdout: true)
        def cid_cs = cid.trim() + ":" + cs.trim()
        def encodeClient = cid_cs.bytes.encodeBase64().toString()
        def tokenCreate = sh(script:"curl -k -d \"grant_type=password&username=admin&password=admin&scope=apim:api_create\" -H \"Authorization: Basic ${encodeClient}\" https://${envPublish}:8243/token | jq -r \'.access_token\'", returnStdout: true)
	def tokenCreateTrimmed = tokenCreate.trim()

	if( api_action.toLowerCase().equals('new')) {	
		def createResponse = sh(script: "curl -k -H \"Authorization: Bearer ${tokenCreateTrimmed}\" -H \"Content-Type: application/json\" -X POST -d @${WORKSPACE}/${API_NAME}.json https://${envPublish}:9443/api/am/publisher/v0.11/apis", returnStdout: true)
                println createResponse
		def createResponseObj = readJSON text: createResponse
		def apiIDCreated = createResponseObj.id
		   if(apiIDCreated != null){
		       apiIDCreated = apiIDCreated.trim()
		   } else {
		       println "API creation failed."
		       sh "exit 1"
		   }
		def tokenPublish = sh(script:"curl -k -d \"grant_type=password&username=admin&password=admin&scope=apim:api_publish\" -H \"Authorization: Basic ${encodeClient}\" https://${envPublish}:8243/token | jq -r \'.access_token\'", returnStdout: true)    
		sh "curl -k -H \"Authorization: Bearer ${tokenPublish}\" -X POST \"https://${envPublish}:9443/api/am/publisher/v0.11/apis/change-lifecycle?apiId=${apiIDCreated}&action=Publish\""	 
       }

       if( api_action.toLowerCase().equals('update')) {
    	      def tokenView = sh(script:"curl -k -d \"grant_type=password&username=admin&password=admin&scope=apim:api_view\" -H \"Authorization: Basic ${encodeClient}\" https://${envPublish}:8243/token | jq -r \'.access_token\'", returnStdout: true)
              def apisList = "["+sh(script:"curl -k -H \"Authorization: Bearer ${tokenView}\" https://${envPublish}:9443/api/am/publisher/v0.11/apis | jq \'.list\' | jq  \'.[] | {id: .id , name: .name , context: .context , version: .version}\'", returnStdout: true)+"]"
	      def formattedJson = apisList.replaceAll("\n","").replaceAll("\r", "").replaceAll("\\}\\{","\\},\\{")
	      def jsonProps = readJSON text: formattedJson
	      int count = 0
	      def updateId
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
	     def json = new JsonSlurper().parse(new File("${WORKSPACE}" + "/" + "${API_NAME}" + ".json"))
	     json << ["id": updateId]
	     new File("${WORKSPACE}" + "/" + "${API_NAME}" + ".json").write(JsonOutput.toJson(json))
	     println JsonOutput.toJson(json)
             json = null //Fix non serialazation exception.
	     def updateResponse = sh(script:"curl -k -H \"Authorization: Bearer ${tokenCreateTrimmed}\" -H \"Content-Type: application/json\" -X PUT -d @${WORKSPACE}/${API_NAME}.json https://${envPublish}:9443/api/am/publisher/v0.11/apis/${updateId}", returnStdout: true)
             println updateResponse
        }
      if( api_action.toLowerCase().equals('delete')) {
    	      def tokenView = sh(script:"curl -k -d \"grant_type=password&username=admin&password=admin&scope=apim:api_view\" -H \"Authorization: Basic ${encodeClient}\" https://${envPublish}:8243/token | jq -r \'.access_token\'", returnStdout: true)
              def apisList = "["+sh(script:"curl -k -H \"Authorization: Bearer ${tokenView}\" https://${envPublish}:9443/api/am/publisher/v0.11/apis | jq \'.list\' | jq  \'.[] | {id: .id , name: .name , context: .context , version: .version}\'", returnStdout: true)+"]"
	      def formattedJson = apisList.replaceAll("\n","").replaceAll("\r", "").replaceAll("\\}\\{","\\},\\{")
	      def jsonProps = readJSON text: formattedJson
	      int count = 0
	      def deleteId
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
