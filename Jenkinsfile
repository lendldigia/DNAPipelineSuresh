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
 
 def file = new File("${WORKSPACE}" + "/" + "${API_NAME}" + ".json")
 
 if(!"${ACTION}".toLowerCase().equals('delete') || ("${ACTION}".toLowerCase().equals('delete') && !file.exists())) {
        stage('CreateMetadataPhase'){     
	String context, versionWithEnv	
	context= "/"+"${API_CTX}"
	pathToTemplate= "${WORKSPACE}" + "/ApiTemplate.json"
	pathToApiMetadata= "${WORKSPACE}" + "/" + "${API_NAME}" + ".json"
	versionWithEnv= "${TARGET_ENV}" + "-" + "${API_VERSION}"
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
	def envt = "${TARGET_ENV}"
        def props = readJSON file: "${WORKSPACE}"+'/Env.json'
        def envPublish = props["${TARGET_ENV}".toLowerCase()]
        println "${API_NAME}"

	def npl = new ArrayList<StringParameterValue>()
	npl.add(new StringParameterValue('TARGET_ENV', envPublish))
	npl.add(new StringParameterValue('API_NAME', "${API_NAME}"))
	npl.add(new StringParameterValue('API_CTX', "${API_CTX}"))
	npl.add(new StringParameterValue('ACTION', "${ACTION}"))
	npl.add(new StringParameterValue('API_DESCRIPTION', "${API_DESCRIPTION}"))
	npl.add(new StringParameterValue('API_VERSION', "${API_VERSION}"))
	npl.add(new StringParameterValue('WSDL_LOC', "${WSDL_LOC}"))
	npl.add(new StringParameterValue('ENDPOINT', "${ENDPOINT}"))
	
	//def build = currentBuild.getRawBuild();
	currentBuild.getRawBuild().replaceAction(new ParametersAction(npl))
        //build = null //Reset state in order to avoid java.io.NotSerializableException.
        //println "${API_DESCRIPTION}"

	if( api_action.toLowerCase().equals('new')) {	
		sh '''echo "**********************************************       Creating clientId and cleintSecret for ADMIN"
		cid=$(curl -k -X POST -H "Authorization: Basic YWRtaW46YWRtaW4=" -H "Content-Type: application/json" -d @payload.json https://${TARGET_ENV}:9443/client-registration/v0.11/register | jq -r \'.clientId\')
		cs=$(curl -k -X POST -H "Authorization: Basic YWRtaW46YWRtaW4=" -H "Content-Type: application/json" -d @payload.json https://${TARGET_ENV}:9443/client-registration/v0.11/register | jq -r \'.clientSecret\')

		encodeClient="$(echo -n $cid:$cs | base64)"
		tokenCreate=$(curl -k -d "grant_type=password&username=admin&password=admin&scope=apim:api_create" -H "Authorization: Basic $encodeClient" https://${TARGET_ENV}:8243/token | jq -r \'.access_token\')

		echo "**************************      CREATING API      ******************************"
		curl -k -H "Authorization: Bearer $tokenCreate" -H "Content-Type: application/json" -X POST -d @${WORKSPACE}/${API_NAME}.json https://${TARGET_ENV}:9443/api/am/publisher/v0.11/apis
		echo "**************************      API CREATED    ******************************"

		echo "**************************      PUBLISHING API      ******************************"
		tokenView=$(curl -k -d "grant_type=password&username=admin&password=admin&scope=apim:api_view" -H "Authorization: Basic $encodeClient" https://${TARGET_ENV}:8243/token | jq -r \'.access_token\')
		tokenPublish=$(curl -k -d "grant_type=password&username=admin&password=admin&scope=apim:api_publish" -H "Authorization: Basic $encodeClient" https://${TARGET_ENV}:8243/token | jq -r \'.access_token\')
		apisList=$(curl -k -H "Authorization: Bearer $tokenView" https://${TARGET_ENV}:9443/api/am/publisher/v0.11/apis | jq \'.list\' | jq  \'.[] | {id: .id , name: .name , context: .context , version: .version}\' )
		createFile=`jq \'{name: .name , context: .context , version: .version}\' ${WORKSPACE}"/${API_NAME}.json"`
		newName="$(echo $createFile | jq -r \'.name\')"
		newContext="$(echo $createFile | jq -r \'.context\')"
		newVersion="$(echo $createFile | jq -r \'.version\')"
		match="$(echo $apisList | jq  --arg creName "$newName" --arg creCon "$newContext" --arg creVer "$newVersion"  \'select((.name==$creName) and (.context==$creCon)  and (.version==$creVer))\')"
		apiIdForPublish="$(echo $match | jq -r \'.id\')"
		curl -k -H "Authorization: Bearer $tokenPublish" -X POST "https://${TARGET_ENV}:9443/api/am/publisher/v0.11/apis/change-lifecycle?apiId=$apiIdForPublish&action=Publish"
		echo "**************************      API PUBLISHED     ******************************"
		'''
          }

        if( api_action.toLowerCase().equals('update')) {	
		sh '''echo "**********************************************       Creating clientId and cleintSecret for ADMIN"
		cid=$(curl -k -X POST -H "Authorization: Basic YWRtaW46YWRtaW4=" -H "Content-Type: application/json" -d @payload.json https://${TARGET_ENV}:9443/client-registration/v0.11/register | jq -r \'.clientId\')
		cs=$(curl -k -X POST -H "Authorization: Basic YWRtaW46YWRtaW4=" -H "Content-Type: application/json" -d @payload.json https://${TARGET_ENV}:9443/client-registration/v0.11/register | jq -r \'.clientSecret\')

		encodeClient="$(echo -n $cid:$cs | base64)"
		tokenCreate=$(curl -k -d "grant_type=password&username=admin&password=admin&scope=apim:api_create" -H "Authorization: Basic $encodeClient" https://${TARGET_ENV}:8243/token | jq -r \'.access_token\')
		tokenView=$(curl -k -d "grant_type=password&username=admin&password=admin&scope=apim:api_view" -H "Authorization: Basic $encodeClient" https://${TARGET_ENV}:8243/token | jq -r \'.access_token\')

		apisList=$(curl -k -H "Authorization: Bearer $tokenView" https://${TARGET_ENV}:9443/api/am/publisher/v0.11/apis | jq \'.list\' | jq  \'.[] | {id: .id , name: .name , context: .context , version: .version}\' )
	
		'echo $envt'
		newName="${API_NAME}"
		newContext="/${API_CTX}"
		newVersion="$envt-${API_VERSION}"
		match="$(echo $apisList | jq  --arg creName "$newName" --arg creCon "$newContext" --arg creVer "$newVersion"  \'select((.name==$creName) and (.context==$creCon)  and (.version==$creVer))\')"

		if [ -n "$match" ]
		then
		echo $match | jq \'{id: .id}\' > updateId.json
		updateId="$(echo $match | jq -r \'.id\')"
		echo $updateId
		echo "**************************      EXECUTING UPDATE      ******************************"
		createFileForUpdate=`jq  -s add updateId.json ${WORKSPACE}"/${API_NAME}.json"`
		

		echo $createFileForUpdate > UpdateAPI.json
		echo "**********************************************       Created file UpdateAPI.json to update API"
		curl -k -H "Authorization: Bearer $tokenCreate" -H "Content-Type: application/json" -X PUT -d @UpdateAPI.json https://${TARGET_ENV}:9443/api/am/publisher/v0.11/apis/$updateId
		echo "**************************      UPDATE COMPLETED      ******************************"
		
		else
		echo "API is not found so update cannot be executed"
		exit 1
		fi
		'''
        }

        if( api_action.toLowerCase().equals('delete')) {	
		sh '''echo "**********************************************       Deleting API ${delApi}"    
		cid=$(curl -k -X POST -H "Authorization: Basic YWRtaW46YWRtaW4=" -H "Content-Type: application/json" -d @payload.json https://${TARGET_ENV}:9443/client-registration/v0.11/register | jq -r \'.clientId\')
		cs=$(curl -k -X POST -H "Authorization: Basic YWRtaW46YWRtaW4=" -H "Content-Type: application/json" -d @payload.json https://${TARGET_ENV}:9443/client-registration/v0.11/register | jq -r \'.clientSecret\')
		encodeClient="$(echo -n $cid:$cs | base64)"
		tokenView=$(curl -k -d "grant_type=password&username=admin&password=admin&scope=apim:api_view" -H "Authorization: Basic $encodeClient" https://${TARGET_ENV}:8243/token | jq -r \'.access_token\')
		apisList=$(curl -k -H "Authorization: Bearer $tokenView" https://${TARGET_ENV}:9443/api/am/publisher/v0.11/apis | jq \'.list\' | jq  \'.[] | {id: .id , name: .name , context: .context , version: .version}\' )


		newName="${API_NAME}"
		newContext="/${API_CTX}"
		newVersion="${TARGET_ENV}-${API_VERSION}"
		match="$(echo $apisList | jq  --arg creName "$newName" --arg creCon "$newContext" --arg creVer "$newVersion"  \'select((.name==$creName) and (.context==$creCon)  and (.version==$creVer))\')"
		
		if [ -n "$match" ]
		then
		deleteId="$(echo $match | jq -r \'.id\')"
		deleteResp=$(curl -k -H "Authorization: Bearer $tokenCreate" -X DELETE https://${TARGET_ENV}:9443/api/am/publisher/v0.11/apis/$deleteId)
		if [ -n "$deleteResp" ]
		then
		echo "**********************************************       Error: $deleteResp"
		fi
		if [ -z "$deleteResp" ]
		then
		echo "**********************************************       API $delApi deleted successfully"
		fi
		
		else
		echo "API is not found so delete cannot be executed"
		exit 1
		fi
                '''
        }
    }
}
