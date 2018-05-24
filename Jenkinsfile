import groovy.io.FileType
import groovy.json.JsonSlurper
import groovy.json.JsonBuilder
import groovy.json.JsonOutput
import hudson.model.* 

node ("bamboor4.dna.fi") {
    stage('CheckoutPhase'){
      checkout scm
    } 

    stage('ValidateParametersPhase'){
	if(!"${ACTION}".toLowerCase().equals('new') && !"${ACTION}".toLowerCase().equals('update') && !"${ACTION}".toLowerCase().equals('delete')){
		println "Invalid Action parameter passed: ${ACTION}"
		       sh "exit 1"		
	} else if (("${ACTION}".toLowerCase().equals('new')) || ("${ACTION}".toLowerCase().equals('update'))){
		if ((!"${API_NAME}"?.trim()) || (!"${API_CTX}"?.trim()) || (!"${API_VERSION}"?.trim()) || (!"${WSDL_LOC}"?.trim()) || (!"${ENDPOINT}"?.trim()) || (!"${TARGET_ENV}"?.trim())){
			println "Mandatory parameter missing: Name= ${API_NAME} , Context= ${API_CTX} , Version= ${API_VERSION} , WSDL= ${WSDL_LOC} , Endpoint= ${ENDPOINT} , Environment= ${TARGET_ENV}"
		       sh "exit 1"
		}
		
	} else if("${ACTION}".toLowerCase().equals('delete')){
	    if ((!"${API_NAME}"?.trim()) || (!"${API_CTX}"?.trim()) || (!"${API_VERSION}"?.trim())){
			println "Mandatory parameter missing: Name= ${API_NAME} , Context= ${API_CTX} , Version= ${API_VERSION}"
		       sh "exit 1"
	    }
	}
	  else {
	      println "Unexpected Validate Error"
		sh "exit 1"
	  }
    }
 
//String envt = "${TARGET_ENV}".toLowerCase()
String context = "${API_CTX}"
String name = "${API_NAME}"
 
if (!"${TARGET_ENV}".toLowerCase().equals('prod')){
  context = context + "-" + "${TARGET_ENV}".toLowerCase()
  name = name + "-" + "${TARGET_ENV}".toLowerCase()
	}

String versionWithEnv= "${API_VERSION}"
String pathToApiMetadata= "${WORKSPACE}" + "/" + "${API_NAME}" + ".json"
if(!"${ACTION}".toLowerCase().equals('delete')) {
        stage('CreateMetadataPhase'){                
          def inputFile = readFile "${WORKSPACE}/ApiTemplate.json"
          def jsonObject = new JsonSlurper().parseText(inputFile)         
          jsonObject.name = name
          jsonObject.context = context
          jsonObject.description = "${API_DESCRIPTION}"
          jsonObject.version = versionWithEnv
          jsonObject.wsdlUri= "${WSDL_LOC}"
          
          def endp = new JsonSlurper().parseText(jsonObject.endpointConfig)
	  endp.production_endpoints.url="${ENDPOINT}"
//	if ("${TARGET_ENV}".toLowerCase().equals('prod')) {
//          endp.production_endpoints.url="${ENDPOINT}"
//		}
//	else {
//		endp.sandbox_endpoints.url="${ENDPOINT}"	
//		}
          String endpointString = JsonOutput.toJson(endp)
          jsonObject.endpointConfig = endpointString          
          jsonString = new JsonBuilder(jsonObject).toPrettyString()  
	  jsonObject = null
    	  endp = null 
          writeFile file: "${WORKSPACE}/${API_NAME}.json", text: jsonString
    }
  }   

   
  
        stage('APIOperationPhase'){
	String api_action = "${ACTION}"
        //def envProps = readJSON file: "${WORKSPACE}"+'/Env.json'
	def readEnvt = readFile "${WORKSPACE}/Env.json"
        //def envProps = new JsonSlurper().parse(new File("${WORKSPACE}"+'/Env.json'))
	def envProps = new JsonSlurper().parseText(readEnvt)
        //String envPublish = envProps["${TARGET_ENV}".toLowerCase()]
		String envPublish= "wso-prod-hel-ui1.s.dnaip.fi"
        envProps = null
        String cid = sh(script: "curl -k -X POST -H \"Authorization: Basic YWRtaW46YWRtaW4=\" -H \"Content-Type: application/json\" -d @payload.json https://${envPublish}:9444/client-registration/v0.11/register | jq -r \'.clientId\'",      returnStdout: true)
        String cs  = sh(script: "curl -k -X POST -H \"Authorization: Basic YWRtaW46YWRtaW4=\" -H \"Content-Type: application/json\" -d @payload.json https://${envPublish}:9444/client-registration/v0.11/register | jq -r \'.clientSecret\'", returnStdout: true)
        String cid_cs = cid.trim() + ":" + cs.trim()
        String encodeClient = cid_cs.bytes.encodeBase64().toString()
        String tokenCreate = sh(script:"curl -k -d \"grant_type=password&username=admin&password=admin&scope=apim:api_create\" -H \"Authorization: Basic ${encodeClient}\" https://wso2-prod.dna.fi/token | jq -r \'.access_token\'", returnStdout: true)
	String tokenCreateTrimmed = tokenCreate.trim()

	if( api_action.toLowerCase().equals('new')) {	
		//String createResponse = sh(script: "curl -k -H \"Authorization: Bearer 201888d9-7888-3dbf-a397-2e8a74a7e4e5\" -H \"Content-Type: application/json\" -X POST -d @${pathToApiMetadata} https://${envPublish}:9444/api/am/publisher/v0.11/apis", returnStdout: true)
		String createResponse = sh(script: "curl -k -H \"Authorization: Bearer ${tokenCreateTrimmed}\" -H \"Content-Type: application/json\" -X POST -d @${pathToApiMetadata} https://${envPublish}:9444/api/am/publisher/v0.11/apis", returnStdout: true)                
		println createResponse
		//def createResponseObj = readJSON text: createResponse
		def createResponseObj = new JsonSlurper().parseText(createResponse)
		String apiIDCreated = createResponseObj.id
                createResponseObj = null
		   if(apiIDCreated != null){
		       println "API created successfully."
		       apiIDCreated = apiIDCreated.trim()
		   } else {
		       println "API creation failed."
		       sh "exit 1"
		   }
		def tokenPublish = sh(script:"curl -k -d \"grant_type=password&username=admin&password=admin&scope=apim:api_publish\" -H \"Authorization: Basic ${encodeClient}\" https://wso2-prod.dna.fi/token | jq -r \'.access_token\'", returnStdout: true)    
		sh "curl -k -H \"Authorization: Bearer ${tokenPublish}\" -X POST \"https://${envPublish}:9444/api/am/publisher/v0.11/apis/change-lifecycle?apiId=${apiIDCreated}&action=Publish\""
		//sh "curl -k -H \"Authorization: Bearer 2117669f-d2ca-388a-a030-79e4809fab34\" -X POST \"https://${envPublish}:9444/api/am/publisher/v0.11/apis/change-lifecycle?apiId=${apiIDCreated}&action=Publish\""	 
       }

       if( api_action.toLowerCase().equals('update')) {
    	      String tokenView = sh(script:"curl -k -d \"grant_type=password&username=admin&password=admin&scope=apim:api_view\" -H \"Authorization: Basic ${encodeClient}\" https://wso2-prod.dna.fi/token | jq -r \'.access_token\'", returnStdout: true)
              def apisList = "["+sh(script:"curl -k -H \"Authorization: Bearer ${tokenView}\" https://${envPublish}:9444/api/am/publisher/v0.11/apis?limit=1000 | jq \'.list\' | jq  \'.[] | {id: .id , name: .name , context: .context , version: .version}\'", returnStdout: true)+"]"
	      def formattedJson = apisList.replaceAll("\n","").replaceAll("\r", "").replaceAll("\\}\\{","\\},\\{")
              println formattedJson
	      //def jsonProps = readJSON text: formattedJson
              def jsonProps = new JsonSlurper().parseText(formattedJson)
              println  "Name is: ${name}, context is: ${context} and version is: ${versionWithEnv}"
              //println jsonProps[0].name
	      int count = 0
	      //String contextCompare = "/${context}"
	      String updateId
              while(count < jsonProps.size()) {
                   //def objAPI = readJSON text: jsonProps[count].toString()
	           //def objAPI = new JsonSlurper().parseText(jsonProps[count].toString())
                   //if("${API_NAME}".toString().equals(objAPI.name) && context.equals(objAPI.context) && versionWithEnv.equals(objAPI.version)){
                   if(name.equals(jsonProps[count].name) && context.equals(jsonProps[count].context) && versionWithEnv.equals(jsonProps[count].version)){
                     updateId =  jsonProps[count].id
                     break
                   }
                   count++
              }
              jsonProps = null
              if(updateId == null){
		       println "Update failed.API ID missing from WSO2."
		       sh "exit 1"
	      }
		def readFile = readFile pathToApiMetadata
	    def json = new JsonSlurper().parseText(readFile)	
 
	     json << ["id": updateId]
		updatedJSON = new JsonBuilder(json).toPrettyString()
	     //new File(pathToApiMetadata).write(JsonOutput.toJson(json))
		json=null
		jsonProps=null
		writeFile file: pathToApiMetadata, text: updatedJSON
	     println JsonOutput.toJson(updatedJSON)
             json = null //Fix non serialazation exception.
	     def updateResponse = sh(script:"curl -k -H \"Authorization: Bearer ${tokenCreateTrimmed}\" -H \"Content-Type: application/json\" -X PUT -d @${pathToApiMetadata} https://${envPublish}:9444/api/am/publisher/v0.11/apis/${updateId}", returnStdout: true)
             println updateResponse
             
        }

      if( api_action.toLowerCase().equals('delete')) {
    	      String tokenView = sh(script:"curl -k -d \"grant_type=password&username=admin&password=admin&scope=apim:api_view\" -H \"Authorization: Basic ${encodeClient}\" https://wso2-prod.dna.fi/token | jq -r \'.access_token\'", returnStdout: true)
              def apisList = "["+sh(script:"curl -k -H \"Authorization: Bearer ${tokenView}\" https://${envPublish}:9444/api/am/publisher/v0.11/apis?limit=1000 | jq \'.list\' | jq  \'.[] | {id: .id , name: .name , context: .context , version: .version}\'", returnStdout: true)+"]"
	      def formattedJson = apisList.replaceAll("\n","").replaceAll("\r", "").replaceAll("\\}\\{","\\},\\{")
	      //def jsonProps = readJSON text: formattedJson
	      def jsonProps = new JsonSlurper().parseText(formattedJson)
	      int count = 0
	      String deleteId
              while(count < jsonProps.size()) {
                   //def objAPI = readJSON text: jsonProps[count].toString()
                   //def objAPI = new JsonSlurper().parseText(jsonProps[count].toString())
                   //if("${API_NAME}".toString().equals(objAPI.name) && context.equals(objAPI.context) && versionWithEnv.equals(objAPI.version)){
		   if("${API_NAME}".toString().equals(jsonProps[count].name) && context.equals(jsonProps[count].context) && versionWithEnv.equals(jsonProps[count].version)){
                     deleteId =  jsonProps[count].id
                     break
                   }
                   count++
              }
              jsonProps = null
              if(deleteId == null){
		       println "Delete failed.API ID missing from WSO2."
		       sh "exit 1"
	      }
            def deleteResponse = sh(script:"curl -k -H \"Authorization: Bearer ${tokenCreateTrimmed}\" -X DELETE https://${envPublish}:9444/api/am/publisher/v0.11/apis/${deleteId}", returnStdout: true)
        }
    }
}

