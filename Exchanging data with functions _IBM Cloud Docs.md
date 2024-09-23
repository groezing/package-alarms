


# Exchanging data with functions

    * Last updated 2024-09-20

Invoking a Code Engine function service (short 'function') is accomplished on the client side through HTTP requests. Information can be passed to the function :

  * as part of the URL (path and optionally, query parameters)
  * through HTTP request headers
  * through the HTTP request body

The result of a function will be received on the client side as an HTTP 
response with response data passed as: 

  * HTTP response code 
  * through HTTP response headers
  * through the HTTP response body
  
All rules and capabilities to invoke a function via HTTP request and the resulting HTTP response are called the "external data interface". 

An incoming HTTP request is forwarded to the user's function code for processing according to the rules of the "internal request data interface". The results provided by the user's function code must follow the rules of the "internal response data interface". 

The following diagram shows the information flow from a client using the external data interface, via the Code Engine functions service and the internal request data interface to the user's function code. Likewise, the result data flow from the user's function code to the internal response data interface via the the Code Engine functions service and the external data interface back to the caller:

 
  ![Functions interfaces.](images/ce_functions_interfaces.svg "Functions interfaces"){: caption="Figure 1. External (caller) data interface and internal (user's function code) data interface" caption-side="bottom"}



## External data interface - Request 

The external data interface definition is derived from the standardized HTTP (Hypertext Transfer Protocol). 
MIME types are used to define the format of the HTTP request information and the HTTP response. 

### MIME types

  The following IANA MIME types are supported for the content type (`Content-Type`) of the HTTP requests or HTTP responses. MIME type parameters are
  supported, but not included in this list:

  * JSON type: 
      * `application/json; <parameters>`
  * Binary types:
      * `audio/*; <parameters>`
      * `application/octet-stream; <parameters>`
      * `example/*; <parameters>`
      * `font/*; <parameters>`
      * `image/*; <parameters>`
      * `model/*; <parameters>`
      * `multipart/*; <parameters>`
      * `video/*; <parameters>`
  * Text types:
      * `text/plain; <parameters>`
      * `text/html; <parameters>`
      * `text/*; <parameters>`
  * Percent-encoded type: 
      * `application/x-www-form-urlencoded; <parameters>`

### Request data 
  When a function is invoked, it can receive arbitrary data (request payload) in text or binary form. The presence of a `Content-Type` in the HTTP request header determines the form and encoding of the data that is sentto the function. The data structure, the format and the content type must match. Code Engine function service runs aminimal validity check on the data and content type and responds with HTTP status code `400` in certain cases, e.g. if the Json data format or encoding is invalid.   
     
  For all other selected `Content-Type` values it is expected that the data format matches the restrictions of the MIME-type. 


#### Providing request data as query paramters
  Request data are provided as key/value pairs in URL encoded format. The IBM Code Engine functions service expects to        get all query parameters of a HTTP request in percent-encoded format. 

  Percent-encoding is widely used in web technology. All 8-bit characters are written as hexadecimal value preceded by a percentage sign (%). This type of encoding is also known as URL-encoding. 
 		
  *Invoke the function by using query parameters:*

```javascript
curl -v "https://sample.1kweru2e873.eu-gb.codeengine.appdomain.cloud/?planet1=Mars&planet2=Jupiter"
```

#### Providing request data in the request body 
		
	Request data are provided in the body section of the HTTP request. The format of the data should match the provided Content-Type. 
		
	*Invoke the function by using body data:* 

```javascript
curl -v -H "Content-Type: application/x-www-form-urlencoded" -d 'location=planet%20earth' https://sample.1kweru2e873.eu-gb.codeengine.appdomain.cloud

curl -v -H "Content-Type: application/json" -d "{'planet1': 'Mars', 'planet2': 'Jupiter'}" https://sample.1kweru2e873.eu-gb.codeengine.appdomain.cloud

WINDOWS:
curl -v  -H "Content-Type: application/json" -d "{\"planet1\":\"Mars\",\"planet2\":\"Jupiter\"}"  https://sample.1kweru2e873.eu-gb.codeengine.appdomain.cloud
```

>Tip: If Content-Type is missing for a request, then Code Engine handles the data payload as described for application/json.
    
        
#### Providing request header data 
	 
Request data are provided as key/value pairs in the header section of the HTTP request.
	
  *Invoke the function by using header fields:*

```javascript
curl -v -H "Sample_Data: Sample_Value"  https://sample.1kweru2e873.eu-gb.codeengine.appdomain.cloud
```
	
#### Providing request mixed data 
	 
The IBM Code Engine functions service allows to combine the usage of different ways to provide request data within one HTTP request. 
		
  *Invoke the function by using body data and header fields:*
	    
```javascript
curl -v -H "Sample_Data: Sample_Value" -H "Content-Type: application/x-www-form-urlencoded" -d 'planet1=Mars&planet2=Jupiter' https://sample.1kweru2e873.eu-gb.codeengine.appdomain.cloud
```
  *Invoke the function by using body data, headers and query parameters:*
```javascript	
curl -v -H "Sample_Data: Sample_Value" -H "Content-Type: application/x-www-form-urlencoded" -d 'planet1=Mars&planet2=Jupiter' https://sample.1kweru2e873.eu-gb.codeengine.appdomain.cloud/?planet1=Mars&planet2=Jupiter
```	


## Internal request data interface 

The internal request data interface describes how the IBM Code Engine functions service delivers the request data received on the external data interface to the user's function code. Independent of the coding language the customer selects to implement its function logic all the request parameters are put to the main() function as the first parameter. The customer can use any name for this parameter, in this documentation its is named "args". 

Additionally the user's function code receives a set of automatically injected pre-defined environment variables 

```javascript	
import os
def main(args):
  raw_input_data = args.__ce_body;
  all_input=args; 		
  env=dict(os.environ)

  return {
    "statusCode": 200,
  }
```


The IBM Code Engine functions service has to use different encodings for the input data to ensure that the user's function code can access the input data in the args argument. The encodings ensure that structure data, 
plain text data and binary data can be part of the input data. 


### Text encoding 

This encoding is used for elements in the input args that contain text payload. The encoding is necessary to ensure that special characters are escaped and do not damage the data structure. 

Code Engine runs the text content encoding before calling the main() function. The function's program logic might need to un-escape them to re-create the original data.

  *Supported escape characters:*

    * Backspace is replaced with |\b|.
    * Form feed is replaced with |\f|.
    * Newline is replaced with |\n|.
    * Carriage return is replaced with |\r|.
    * Tab is replaced with |\t|.
    * Double quotation marks are replaced with |\"|.
    * Backslash is replaced with |\\|.

```
    "###\nretu\\rned text body\f###\n"
```
		
#### Base64 encoding 
	
This encoding is used for elements in the input args that could break the internal data structure of the args parameter. 
By encoding it as base64 it is ensured that the whole value consist only of readable characters in a 64-character alphabet. 
	
```
  "eyAicGxhbmV0MSI6Ik1hcnMiLCAicGxhbmV0MiI6Ikp1cGl0ZXIiIH0="
```
	
	
#### Percent encoding (Url-encoding) 
	 
Percent-encoding is widely used in web technology. All 8-bit characters are written as hexadecimal value preceded by a percentage sign (%). This type of encoding is also known as URL-encoding. The function receives the form-urlencoded type data in raw format in the __ce_body parameter.
			 	 
The IBM Code Engine functions service is passing the query parameters as-is (the leading |?| is not part of it) into the __ce_query field of the args parmameter. 
So the user's function code will receive this parameter in URL-encoded format. 
	
```
  "x%5cb=1%22f4%20and%20"
```

	
### The args parameter  
  
The args parameter is a language specific data structure (JSON object for Node.js and dict type for Python) filled with the request input data in following format: 

| top-level elements | description |
|--------------------|-------------|
| \_\_ce_....        | ce_functions input argument   | 
| \_\_ce_headers     | ce_header is a copy of the HTTP request header   |
| \<any\> (optional)   | top-level properties available if the HTTP request input data are provided in structured format (only if Content-Type 'application/json' is used)  |
	
```javascript
  {
    "args": {
      "__ce_body": "eyAicGxhbmV0MSI6Ik1hcnMiLCAicGxhbmV0MiI6Ikp1cGl0ZXIiIH0=",
      "__ce_headers": {
        "Accept": "*/*",
        "User-Agent": "curl/7.58.0",
        "X-Request-Id": "d03a1af0-bfc8-4a50-be7d-a72040c02cc9",
        "Content-Type": "application/json",
      },
      "__ce_method": "GET",
      "__ce_path": "/",
      "__ce_query": "",
      "planet1": "Mars",
      "planet2": "Jupiter"
      }
  }
```

#### Args parameter set for internal input arguments (\_\_ce_*)
	
The IBM Code Engine functions service injects reserved input arguments to the args input data.
The following parameters can be found in a Code Engine function's input arguments:

| parameter name     | value type        | description |
|--------------------|-------------------|-------------|
| __ce_method        | String            | The HTTP method of the request (GET/POST) |
| __ce_headers       | Map of key/value  | key/value pairs of HTTP request header |
| __ce_path          | String            | Url path of the incoming HTTP request  |
| __ce_body          | String            | The request body entity, as a Base64-encoded string when request content type is binary, or as a plain string, otherwise  |
| __ce_query         | String            | The query parameters from the request as an unparsed string. The __ce_query parameter is a single string that contains the query parameters that are parsed from the URL, without the leading question mark (?), and separated by an ampersand (&).  |


The internal \_\_ce_* parameters cannot be overwritten with request data provided int the HTTP request. HTTP calls trying to overwrite will fail with status 400 (Bad Request).

	
#### Args parameter from query paramters of function invocation 
	  
The function receives the query parameter data as key/value in the __ce_query parameter as percent-encoded (url-encoded) and additionally each single query parm as key/value pair with in decoded format. 
	 
| args field         | is set ?          | description |
|--------------------|-------------------|-------------|
| __ce_body          | NO                |  |
| __ce_headers       | YES               | but no content-type |
| __ce_query         | YES               | query parameter (url-encoded) |
| top-level property | YES               | key/value  (url-decoded and text-encoded)  |

	   
*Python example Accessing query parameters* 
```javascript     
import os
def main(args):
  query_parm_1 = args.get("key", "default_value") # get the value of one query parameter 
  
  try:
    query = args["__ce_query"]  # get all query parms 
  except:
    query= "not part of request"
 
  return {
    "statusCode": 200,
    "body": query,
  }
```

  
*Nodejs example Accessing query parameters* 
```javascript     
function main(args) {
  var query_parm_1 = args.key     // get the value of one query parameter 
  var query = args.__ce_query     // get all query parameters 
   
  return {
    statusCode: 200,
    "body" : query,
  };
}


module.exports.main = main;
```    

	
#### Args parameter from header data of function invocation 

The function receives the request header data as key/value in the __ce_headers section. The key value will be converted into canonical format, independent in which format the key is set on the external data interface (e.g. "mykey" or "MYKEY" is converted to "Mykey")

| args field         | is set ?          | description |
|--------------------|-------------------|-------------|
| __ce_body          | NO                |             |
| __ce_headers       | YES               | Key/value (text-encoded) (key is in canonical format) |
| __ce_query         | ""                | empty string |
| top-level property | NO                |   |

*Python example Accessing header data* 
   
```javascript
  import os
  def main(args):

    try:
      header = args["__ce_headers"]  # get complete header 
      # value_1 = args["__ce_headers"]["Key_1"]  # get value of the Header parm with "Key_1" 
    except:
      header = "not part of request"
 
    return {
      "statusCode": 200,
      "body": header,
    }
```
*Nodejs example accessing header data* 
   
```javascript   
function main(args) {
  // var header_parm_1 = args.__ce_headers.Key_1  //get the value of one header parameter
  var header = args.__ce_headers  // get all header parameters
  
  return {
    statusCode: 200,
    "body" : header,
  };
}

module.exports.main = main;
```





#### Args parameter from request data of Content-type "application/json" 
	
The function receives the keys and values of the JSON document as dedicated top-level property parameters.
If no Content-type is set in the HTTP request then IBM Code Engine function service uses "application/json" as default. 

In addition, the JSON payload is made available to the function as-is (as byte array) in Base64-encoded format in the |__ce_body| parameter.
	  
| args field         | is set ?          | description |
|--------------------|-------------------|-------------|
| __ce_body          | YES               | request data (base64-encoded) |
| __ce_headers       | YES               | content-type is set |
| __ce_query         | ""                | empty string |
| top-level property | YES               | key/value for each top-level element (text-encoded)   |


*Python example accessing application/json input data* 
```javascript   
  import os
def main(args):
  try:
    body_encoded = args["__ce_body"]  # get complete header (base64 encoded)
    value_1 = args["key_1"]           # get value of the Header parm with "key_1" 
  except:
    value_1 = "not part of request"
 
  return {
    "statusCode": 200,
    "body": value_1,
  }
```   


*Nodejs example accessing application/json input data* 
```javascript   
function main(args) {
  var body  = args.__ce_body  // get complete request body (base64 encoded)
  var value_1 = args.key_1    // get value of one single key 
  
  return {
    statusCode: 200,
    "body" : value_1,
  };
}

module.exports.main = main;
``` 

#### Args parameter from request data of Content-type "application/octet-stream" 

The function receives the binary data in Base64-encoded format in the `__ce_body` parameter

| args field         | is set ?          | description |
|--------------------|-------------------|-------------|
| __ce_body          | YES               | request data (base64-encoded) |
| __ce_headers       | YES               | content-type is set |
| __ce_query         | ""                | empty string |
| top-level property | NO                |   |
	
*Example Accessing application/octet-stream input data* 
  
```javascript     
  import os
	import base64

  def main(args):
    body_encoded = args.__ce_body;
		body = binaryRequestBody = base64.b64decode(args['__ce_body'])
		 
    return {
      "statusCode": 200, 
    }  
```
 
#### Args parameter from request data of Content-type  "text/plain" 
	
The function receives the text type data in Base64-encoded format in the __ce_body parameter.
The function's program logic might need to unescape them to recreate the original data.
	  

| args field         | is set ?          | description |
|--------------------|-------------------|-------------|
| __ce_body          | YES               | request data (text-encoded) |
| __ce_headers       | YES               | content-type is set |
| __ce_query         | ""                | empty string |
| top-level property | NO                |   |
	
	  
*Example Accessing text/plain input data* 
```javascript     
  import os
  def main(args):
    body_escaped = args.__ce_body;
		 
    return {
      "statusCode": 200, 
    }
```     
  
#### Args parameter from request data of Content-type "application/x-www-form-urlencoded" 
	
The function receives the complete body data in text-encoded format in the __ce_body parameter.
	 
| args field         | is set ?          | description |
|--------------------|-------------------|-------------|
| __ce_body          | YES               | request data (text-encoded) |
| __ce_headers       | YES               | content-type is set |
| __ce_query         | ""                | empty string |
| top-level property | NO                |   |
	  
	   
*Example Accessing application/x-www-form-urlencoded input data* 
```javascript  
  import os
  def main(args):
    body_escaped = args.__ce_body;
		
    return {
      "statusCode": 200, 
    }
``` 
  
#### Args parameter from request data of Content-type "application/x-www-form-urlencoded" and query params 
	
All combinations of mixed data types are possible, but following rules must be considered: 
	 
For HTTP requests with URL query parameters and body parameters the body parameters take precedence over query parameters.
	 

###  Code Engine function environment variables
  
While function call parameters are derived from the incoming HTTP request, a Code Engine function also has access	to a set of predefined environment variables derived from system settings:

  * CE_ALLOW_CONCURRENT
  * CE_API_BASE_URL
  * CE_DOMAIN
  * CE_EXECUTION_ENV
  * CE_FUNCTION
  * CE_PROJECT_ID
  * CE_REGION
  * CE_SUBDOMAIN

For more information about these environment variables, see <link> Automatically injected environment variables.

*Example Accessing the environment variables* 

```javascript     
	import os

	def main(args):
    curEnv=dict(os.environ)
		return {
			"headers": { "Content-Type": "application/json" },
			"statusCode": 200,
			"body": {
		   	"env": curEnv,
			}
		}
```    


## Internal response data interface 

The internal response data interface depicts how a user's function code provides response data. The Code Engine function service consumes them and provides it as HTTP response to the caller. 
Independent of the programming language the user's function code has to provide the response data as data structure in the return statement. 
Dependent of the programming language the result object is either a Json object (Node.js language) or a dictionary (Python language) with the following structure: 

| user functios result data |                      | description                     |
|---------------------------|----------------------|---------------------------------|
| headers                   | Optional             | A result object in which the keys are header names and the values are strings, numbers, or Boolean values. To send multiple values for a single header, the header's value is an array of the multiple values. No headers are set by default |
| statusCode                | Should (default 200) | A valid HTTP status code |
| body                      | Optional (default: empty) | A string that is either plain text, a JSON object or array, or a Base64 encoded string for binary data. The body is considered empty if it is null, an empty string (""), or undefined.|

*Example result data structure to provide function execution result data* 

```javascript     
  import os

  def main(args):
	  return {
      headers: { 
        "content_type" : "application/json" , 
		    "key" , "value" 
      },
      "statusCode": 200,
		  body: {"myMessage" : "sample message"}
    }
```


>Note : It is recommended to set body and statusCode always. The implicit setting of statusCode and body is only available for compatibility to IBM Cloud Functions deprecated service. 


### The headers element
  
 The main intention of the headers section is to define the content_type in which the user's function is going to provide the response data. Result data supports the identical MIME-Types as accepted on request. 
 
Depending on the value of the Content-Type the body field in the result structure has to contain the following:


| Content type       | value assign to body element | Example                     |
|--------------------|------------------------------|-----------------------------|
| application/json   | JSON object                  | body : { key_1: val1 }      |
| Not set (defaul)   | string                       | body :  "some text"         |
| text/*             | string                       | body :  "some text"         |
| audio/*, example/*, font/', image/*, video/*, and all remaining types | base64 encoded | body: "SGVsbG8gV29ybGQhCg==" |
  
  
Additionally the user function code may use key/value pairs in the HTTP response header to provide response data. Therefore it must put a unique key name and its value into the headers section of the data structure used in the return statement. 
  
**Keynames must consider:** 
  
  * keynames are case-insensitive
  * latest value assignment of identical keynames is used 
  * backslashes or blanks are not supported
	
**Values must :** 
  * must be text encoded (see text_encoding chapter) 
  * must be of data type string or array of string
	
*Example returning response data as Header key/value* 
```javascript      
import os
  def main(args):
        
  return {
    "headers": {
      "Content-Type": "text/plain",
	  	"key_1" : "sample_value",
    },
    "statusCode": 200,
    "body": "" , 
	}

```     
	
 
 ### The statusCode element
  
 The user function code should explicitly set a status code (default 200) to inform the caller about the execution result. Any valid HTTP RC status code may be used ( 200 <= statusCode < 600 ).
 IBM Code Engine function service is returning the user function's status code in the header field (x-faas-actionstatus) and as HTTP status code. 
 
 If the statusCode is invalid then the IBM Code Engine functions service return the HTTP RC = 422 ("Invalid function code cannot be processed") without 
 the x-faas-actionstatus header field and without additional response data.  
 If the result size limit for functions is reached, an HTTP status code |400| is returned to the client.
 
*Example returning response status code* 
```javascript     
import os
  def main(args):
        
  return {
    "statusCode": 200,
    "body": "" , 
	}

```     
 
### The body element
  
The user function code may use the body section of the return data structure to provide the user function's response data. The function code is responsible to deliver the body element in the return data structure in the content-type matching format. If the function response `Content-Type` is missing then the function code returns `Content-Type: text/plain; charset=utf-8` at the external interface, and the data from the `body` field in its unaltered form.
   
Therefore the following rules must be considered depending on the content type.  

#### body value of Content-type "application/json" 
	
The user function code must provide the value of the body element as a valid data structure (NodeJs-Json or Python-Dictionary). The keys and values of the data structure must follow the Json Syntax rule. That enables the IBM Cloud Code engine service to deliver the response data as a valid application/json HTTP Response data section. 
	   
>Note: backslashes(\) and double quotes(") in the data strucuture will be delivered in text-encoded format in the HTTP Response data section. 

*Example python application/json response* 
```javascript     
  import os
  def main(args):
    # python dictionary 
	  result_body = { "key_1" : "myfolder\myFile" }
		
    return {
      "headers": {
        "Content-Type": "application/json",
      },
      "statusCode": 200,
      "body": result_body, 
		}
```
   
*Example NodeJs application/json response* 
```javascript     
  function main(args) {
    // JSON data structure  
    // Note: Backslash must be text encoded 
	  result_body =  { "key1" : "myfolder\\myFile"}
	  
	  return {
		  statusCode: 200,
		  headers: { 
		    'Content-Type': "application/json", 
		  },
		  "body" : result_body ,
	  };
	} 
  module.exports.main = main;
```

#### body value of Content-type "application/octet-stream" 
	
The user function code has to base64 encode the result data before adding to the result structure. 

*Example Python application/octet-stream response* 
```javascript 	   
	import os
	import base64
	  
  def main(args):
    result_body = "myfolder_myFile"
    enc_result_body=base64.encodebytes(result_body.encode()).decode("utf-8").strip()
		
    return {
      "headers": {
        "Content-Type": "application/octet-stream",
      },
      "statusCode": 200,
      "body": enc_result_body, 
    }
```	   

*Example NodeJs application/octet-stream response* 
```javascript 	   
  function main(args) {
    var result_body = "###\unreturned body###\n"
    var buff = new Buffer(result_body , 'utf8');
		 
    return {
      statusCode: 200,
      headers: { 
       'Content-Type': "application/octet-stream", 
      },
      "body" : buff.toString('base64') ,
    };
  }

  module.exports.main = main; 
```	   
	  
#### body value of Content-type "text/plain" 
	
The user function code provides the whole response in one single string. The IBM Code Engine functions service is not checking the response data in any way. The data are transfered to the caller as provided. 
	   
*Example Python text/plain response* 
```javascript 	   
  def main(args):
    result_body = "myfolder_myFile"

    return {
      "headers": {
        "Content-Type": "text/plain;charset=utf-8",
      },
      "statusCode": 200,
      "body": result_body, 
    }
```


#### body value of Content-type "application/x-www-form-urlencoded" (Percent-encoded content) 
	
The user function code has to ensure that the response data are url-encoded before being added to the result data structure. The IBM Code Engine functions service is not checking the response data for correct syntax. The data are transfered to the caller as provided. 
	
*Example Python application/x-www-form-urlencoded response* 
```javascript 	   
  import os
  def main(args):
    result_body = "myfolder%20myFile"

    return {
      "headers": {
      "Content-Type": "application/x-www-form-urlencoded",
      },
      "statusCode": 200,
      "body": result_body,  
    }
```	  
     

## External data interface - Response 

The external data interface definition is derived from the standardized HTTP (Hypertext Transfer Protocol). MIME types in the HTTP Response header are used to define the format of the HTTP response. 


### MIME types
When Code Engine returns data to the external world, Code Engine functions can support the same IANA MIME types as described in MIME 	types <https://cloud.ibm.com/docs/codeengine?topic=codeengine-fun-exchanging-data#fun-external-data-interface-MIME-types>.

### Response data 
  
The external data interface is returning the response data structure as  HTTP header, HTTP status code and HTTP response data to the caller of the function. 
	
	
#### The HTTP Header fields 

The HTTP header contains all key/value pairs the user's function code added to the result data structure. 
Beneath that IBM Code Engine functions service adds following key/value pairs: 
| Field name           | description |
|----------------------|-----------------------------------------------------------|
| x-request-id         | The external request id for the function invoke      |
| x-faas-activation-id |  The IBM Code Engine functions service internal ID for the invocation        |
| x-faas-actionstatus  | The status code the user's function code set        |

	
>Note: header key names are always replied as lower-case
	
#### The HTTP status code 

The origin of the HTTP status code can be the IBM Code Engine functions service itself or the status code set by the user's function code. The presence of the header field "x-faas-statuscode" is the indicator of the origin. 

The meaning of the HTTP status code can be read in the IBM Code engine functions documentation if the header field "x-faas-actionstatus" is not set. But if the header field is set, then the meaning of the HTTP status code is defined by the logic of the user's function. 

>Note: IBM Code Engine functions service  runs a limited validity check on the response data and content type, and returns HTTP status code 400 if the data format or             encoding (generated by the user's function code) is invalid.


#### The HTTP Response data  

The HTTP response data on the external data interface are identical as supplied in the result data structure by the user's function code.  Only if the MIME-Type is "application/json" then the key and values in the provided json response are modified. All backslashes and double quotes are escaped. 

*Example How the caller get application/json response data * 

__User function code providing response data__
```javascript 	    
  import os
  def main(args):
    # python dictionary 
    result_body = { "key_1" : "myfolder\myFile" }
  
    return {
      "headers": {
      "Content-Type": "application/json",
      "key" : "sample",
      },
      "statusCode": 200,
      "body": result_body, 
    }
```		  
__curl client consuming the response status_code, the header fields and the response data__
```javascript 	    
  curl -v -i  https://sample.1kweru2e873.eu-gb.codeengine.appdomain.cloud/  
	 -> Response: 
	stdout> 
   * Host default-encoded-response.1lpigeajq5fg.us-east.codeengine.appdomain.cloud:443 was resolved.
   * IPv6: 2606:4700:90:0:7554:9304:2cbe:8cbf
   * IPv4: 172.65.197.223
   * using HTTP/1.x
   > GET / HTTP/1.1
   > Host: default-encoded-response.1lpigeajq5fg.us-east.codeengine.appdomain.cloud
   > User-Agent: curl/8.8.0
   > Accept: *//*
 
   * Request completely sent off
   < HTTP/1.1 200 OK
   < content-type: application/json
   < key: sample
   < x-faas-actionstatus: 200
   < x-faas-activation-id: 5cbab12c-5c6e-4000-96cf-0f7fcb42a979
   < x-request-id: e7098271-4780-4893-bbd4-64d4c8d7605e
   < content-length: 13
   { "key_1" : "myfolder\\myFile }* Connection 
 ```



[Info from FG: Docu instruction meta data. Re-use the invocation examples as Is from current docu version ]
##  Code Engine function invocation examples

Follow these steps to see the creation of a Code engine function with the Command line interface (CLI) and the usage of the external data interface.

 1.

    Save the following code as |hello.js| locally 

```javascript
  function main(args) {
    return { headers: { content_type: "application/json" },
      statusCode: 200,
    	body: {args: args}
    };
  }

```
 2.

    After logon to IBM Cloud and selection of the IBM Code Engine functions service, Create a new function with the code:

    |ibmcloud ce fn create --name sample --runtime nodejs-18 --inline-code sample.js --cpu 0.5 --memory 2G

    |

    Example output:

    |Creating function 'sample'...
    OK
    Run 'ibmcloud ce function get -n sample' to see more details.

    # ibmcloud ce function get -n sample
    Getting function 'sample'...
    OK

    Name:    sample
    ...
    Status:  Ready
    URL:     https://sample.1kweru2e873.eu-gb.codeengine.appdomain.cloud
    ...

    |

 3.

    Use the external data interface with a curl command to invoke the function:

    |curl -v https://sample.1kweru2e873.eu-gb.codeengine.appdomain.cloud/ | jq .

    |

    Example output:

    |{
        "args": {
            "__ce_headers": {
            "Accept": "*/*",
            "User-Agent": "curl/7.58.0",
            "X-Request-Id": "813804ec-ef14-42e9-bce3-c162373defae"
            },
            "__ce_method": "GET",
            "__ce_path": "/"
        }
    }

    |

 4.

    Invoke the function by using query parameters:

    |curl -v "https://sample.1kweru2e873.eu-gb.codeengine.appdomain.cloud/?planet1=Mars&planet2=Jupiter" | jq .

    |

    Example output:

    |{
        "args": {
            "__ce_headers": {
            "Accept": "*/*",
            "User-Agent": "curl/7.58.0",
            "X-Request-Id": "d03a1af0-bfc8-4a50-be7d-a72040c02cc9"
            },
            "__ce_method": "GET",
            "__ce_path": "/",
            "__ce_query": "planet1=Mars&planet2=Jupiter",
            "planet1": "Mars",
            "planet2": "Jupiter"
        }
    }

    |

    The query parameter is available unfolded in the arguments and also
    unmodified in |__ce_query|.

 5.

    Invoke the function by using form data:

    |curl -H "Content-Type: application/x-www-form-urlencoded" -d 'planet1=Mars&planet2=Jupiter' https://sample.1kweru2e873.eu-gb.codeengine.appdomain.cloud | jq .

    |

    Example output:

    |{
        "args": {
            "__ce_body": "planet1=Mars&planet2=Jupiter",
            "__ce_headers": {
            "Accept": "*/*",
            "Content-Length": "28",
            "Content-Type": "application/x-www-form-urlencoded",
            "User-Agent": "curl/7.58.0",
            "X-Request-Id": "eb3e2179-c396-4eee-98cb-809316f0a765"
            },
            "__ce_method": "POST",
            "__ce_path": "/"
        }
    }

    |

    The content of the request body is available unmodified in the |__ce_body| argument with JSON reserved characters being escaped.

 6.

    Invoke the function by using a JSON data object:

    |curl -H "Content-Type: application/json" -d '{"planet1": "Mars", "planet2": "Jupiter"}' https://sample.1kweru2e873.eu-gb.codeengine.appdomain.cloud | jq .

    |

    Example output:

    |{
        "args": {
            "__ce_body": "eyJwbGFuZXQxIjogIk1hcnMiLCAicGxhbmV0MiI6ICJKdXBpdGVyIn0=",
            "__ce_headers": {
            "Accept": "*/*",
            "Content-Length": "41",
            "Content-Type": "application/json",
            "User-Agent": "curl/7.58.0",
            "X-Request-Id": "c06ffcc3-fdae-4430-b881-01d68876c54c"
            },
            "__ce_method": "POST",
            "__ce_path": "/",
            "planet1": "Mars",
            "planet2": "Jupiter"
        }
    }

    |

    The content of the request body is available unfolded into the
    function arguments (|args|) with JSON reserved characters being escaped and also unmodified
    as a Base64-encoded string in |__ce_body|.

 7.

    Invoke the function by using JSON data and query parameters:

    |curl -H "Content-Type: application/json" -d '{"planet1": "Mars", "planet2": "Jupiter"}' "https://sample.1kweru2e873.eu-gb.codeengine.appdomain.cloud?planet2=Venus&planet3=Uranus" | jq .

    |

    Example output:

    |{
        "args": {
        "__ce_body": "eyJwbGFuZXQxIjogIk1hcnMiLCAicGxhbmV0MiI6ICJKdXBpdGVyIn0=",
            "__ce_headers": {
            "Accept": "*/*",
            "Content-Length": "41",
            "Content-Type": "application/json",
            "User-Agent": "curl/7.58.0",
            "X-Request-Id": "daff83a5-fe53-43ef-8dc4-606e42dd8306"
            },
            "__ce_method": "POST",
            "__ce_path": "/",
            "planet1": "Mars",
            "planet2": "Jupiter",
            "planet3": "Uranus"
        }
    }

    |

    The body parameters and query parameters are unfolded into the
    function arguments (|args|). Body parameters overwrite the query parameters. The request body
    is available in |__ce_body| (Base64-encoded). The query parameters are available in |__ce_query|.

 8.

    Invoke the function by using text content type:

    |curl -H "Content-Type: text/plain" -d 'Here we have some text. The JSON special characters like \ or " are escaped.' https://sample.1kweru2e873.eu-gb.codeengine.appdomain.cloud | jq .

    |

    Example output:

    |{
        "args": {
            "__ce_body": "Here we have some text. The JSON special characters like \\ or \" are escaped.",
            "__ce_headers": {
            "Accept": "*/*",
            "Content-Length": "76",
            "Content-Type": "text/plain",
            "User-Agent": "curl/7.58.0",
            "X-Request-Id": "43d259eb-4247-41a9-894a-1dbd98eb16fb"
            },
            "__ce_method": "POST",
            "__ce_path": "/"
        }
    }

    |

    The content of the request body is available unmodified in |__ce_body|, but with JSON special characters escaped with |''|.

 9.

    Invoke the function by using binary content type:

    |curl -H "Content-Type: application/octet-stream" -d 'This string is treaded as binary data.' https://sample.1kweru2e873.eu-gb.codeengine.appdomain.cloud | jq .

    |

    Example output:

    |{
        "args": {
            "__ce_body": "VGhpcyBzdHJpbmcgaXMgdHJlYWRlZCBhcyBiaW5hcnkgZGF0YS4=",
            "__ce_headers": {
            "Accept": "*/*",
            "Content-Length": "38",
            "Content-Type": "application/octet-stream",
            "User-Agent": "curl/7.58.0",
            "X-Request-Id": "a90826e0-db13-4b8a-809f-60cea2a27d96"
            },
            "__ce_method": "POST",
            "__ce_path": "/"
        }
    }

    |


        Decoding binary body content from Base64

You can decode a function's binary body content from Base64 as follows.

Example of decoding functions in Node.js:

|function main(args) {
    binaryRequestBody = new Buffer(args.__ce_body, 'base64').toString('utf-8')

	// The buffer binaryRequestBody now contains a copy of the request body.
	....

}

|

Example of decoding functions in Python:

|import base64

def main(args):
    try:
        binaryRequestBody = base64.b64decode(args['__ce_body'])
    except:
        return {
            "body": "Error decoding body from Base64.",
            "statusCode": 400
        }

	# The byte array binaryRequestBody now contains a copy of the request body.
	...

|
































