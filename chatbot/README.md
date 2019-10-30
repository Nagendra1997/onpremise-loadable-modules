#  Chatbot Module


This repository contains the source code for Chatbot Module. Chatbot module analyzes messages using various AI and machine learning tools like Tensorflow, Dialogflow, etc. and sends an automatic reply.  

To build chatbots, you can use any chatbot provider of your choice such as Dialogflow, IBM Watson,Rasa,etc and connect with them through REST endpoints. 
This Chatbot Module provides an example using **Dialogflow**

- You can download the source and compile it to obtain the module- a shared library file. 

- The complete documentation for Mesibo Modules is available here

- Refer to the [Skeleton Module](https://github.com/mesibo/onpremise-loadable-modules/tree/master/skeleton) for a basic understanding of how Mesibo Modules work. The complete documentation for Mesibo Modules is available [here](https://mesibo.com/documentation/loadable-modules/)


## Overview of the Chatbot Module
- Chatbot configuration containing :project name, session id, Endpoint/base url for making REST call, access token is provided in the module configuration file (Refer `sample.conf`).
- In module initialisation, all the configuration parameters are obtained and stored in a structure object `chatbot_config_t`. 
- The callback function for `on_message` is initialized to  `chatbot_on_message`
- When a message is sent from a user to the chatbot `address`, Mesibo invokes the `chatbot_on_message` callback function of the module with the message data and message parameters.
- Now, the chatbot module makes an HTTP request to a REST endpoint of Dialogflow .The HTTP Post body contains the message text . The authorisation header containing the access token is also sent in the POST request.
- Dialogflow sends a response , which is recieved through an HTTP callback function.
- The response text from the chatbot is extracted from the JSON response and is sent to the user (sender)

### Basics of Dialogflow
Dialogflow is an AI powered Google service to build interactive conversational interfaces, such as chatbots. Once you train DialogFlow using data of your interest like emails, FAQs, etc., it can answer questions from your users in natural language.

Dialogflow service is available through a REST interface. For more details on using Dialogflow , refer [DialogFlow Documentation](https://cloud.google.com/dialogflow/docs/quick/api)

#### Configuring Dialogflow API 
To use Dialogflow API, you will need two parameters which you can obtain from the Google Cloud Console.

  - GCP project ID
  - Access Token

Following are the the steps:

1. Set up a [GCP Console](https://console.cloud.google.com) project.
    - Create or select a project and note the project ID
    - Create a service account
    - Download a private service account key as JSON

2. Set the environment variable `GOOGLE_APPLICATION_CREDENTIALS` pointing to the JSON file downloaded in the Step 1.

```
export GOOGLE_APPLICATION_CREDENTIALS="/home/user/Downloads/service-account-file.json"
```

3. [Install and initialize the Cloud SDK](https://cloud.google.com/sdk/docs/)
4. Print your access token by using the following command

```
echo $(gcloud auth application-default print-access-token)
```

which should output something like

```
ya29.c.Kl6iB-r90Gjj4o--m7k7wr4dN4b84U4TLEtPqdEZ2xvfsj01awmUObMDEFwJIJ1lkZPym5dsAw44MbZDSaksLH3xKbsSHGLgWeEXqIPSDmFO6
```
This is the access token, save it for later use.

#### Invoking Dialogflow API  
Once we have project ID and access token, invoking DialogFlow API is as simple as invoking following URL with access token and the data:

```
https://dialogflow.googleapis.com/v2/projects/<Project ID>/agent/sessions/<Session ID>
```

Where, Project ID is the GCP Project ID obtained earlier. Session ID can be a random number or some type of user and session identifiers (preferably hashed). 

For example, a sample dialogflow REST URL looks like

```
https://dialogflow.googleapis.com/v2/mesibo-dialogflow/agent/sessions/123456789
```

Now, you send a POST request to the above URL in the following format.

Pass the authentication information in the request header.

```
Authorization: Bearer <YOUR_ACCESS_TOKEN>
Content-Type: application/json
```

and the POST data which contains your text/message.

```
{
    "queryInput": {
 	{
	"text": {"text": <message text>}
	}   
    }
}
```

### Configuring the chatbot module
While loading your module you need to provide your configuration details in the file `/etc/mesibo/mesibo.conf`. The sample configuration is provided in the file `sample.conf`. 
The chatbot module is configured as follows:

```
module chatbot{
project = <Project Name>
session = <Session ID>
endpoint = <Dialogflow REST Endpoint>
access_token = <Service Account key>
chatbot_uid = <Chatbot User ID>
log = <log level>
}
```

where, 
- `project`, GCP Project ID that contains your dialogflow chatbot.
- `session`, The Session ID for the queries being sent . 
- `endpoint`, The dialogflow REST endpoint to which your query will be sent
- `access_token`, access token linked with your project. In this example we will be passing dialogflow client access token in the auth header like so `Authorisation: Bearer <access token>.`
- `address`, chatbot address. In your Mesibo Application, create a user that you can refer to as a chatbot endpoint user. 
- `log`, Log level for printing to mesibo container logs


For example,
```
module chatbot{
project = mesibo-chatbot
session = 4e72c746-7a38-66b6-xxxxxx
endpoint = https://dialogflow.googleapis.com/v2beta1
access_token = x.Kl6iBzVH7dvV2XywzpnehLJwczdClfMoAHHOeTVNFkmTjqVX7VagKH1
chatbot_uid = my_chatbot
log = 1
}
```

### 3. Initialization of the chatbot module
The chatbot module is initialized with the module description and  references to the module callback functions.
```cpp

int mesibo_module_chatbot_init(mesibo_module_t *m, mesibo_uint_t len) {

        MESIBO_MODULE_SANITY_CHECK(m, version. len);

        m->flags = 0;
        m->description = strdup("Sample Chatbot Module");
        m->on_message = chatbot_on_message;

        if(m->config) {
                chatbot_config_t* cbc = get_config_dialogflow(m);
                if(cbc  == NULL){
                        mesibo_log(m, MODULE_LOG_LEVEL_0VERRIDE, "%s : Missing Configuration\n", m->name);
                        return MESIBO_RESULT_FAIL;
                }
                m->ctx = (void* )cbc;

                int init_status = chatbot_init_dialogflow(m);
                if(init_status != MESIBO_RESULT_OK){
                        mesibo_log(m, MODULE_LOG_LEVEL_0VERRIDE, "%s : Bad Configuration\n", m->name);
                        return MESIBO_RESULT_FAIL;
                }
        }

        else {
                mesibo_log(m, MODULE_LOG_LEVEL_0VERRIDE, "%s : Missing Configuration\n", m->name);
                return MESIBO_RESULT_FAIL;
        }

	return MESIBO_RESULT_OK;
```

First a sanity check is performed using `MESIBO_MODULE_SANITY_CHECK` and then the configuration is retrieved and stored in the module context m->ctx so that it is available whenever module callbacks are invoked.
The configuration structure chatbot_config_t is used to store the configuration:

```cpp
typedef struct chatbot_config_s {
        /* To be configured in module configuration file */
        const char* project;
        const char* session;
        const char* endpoint;
        const char* access_token;
        const char* chatbot_uid;
        int log;

        /* To be configured by Dialogflow init function */
        char* post_url;
        char* auth_bearer;
        module_http_option_t* chatbot_http_opt;

} chatbot_config_t;
```

To get the configuaration information, the config helper function `get_config_dialogflow` is called.

`get_config_dialogflow` fetches all the configuration details as per format and stores into the appropriate members of the structure `chatbot_config_t`

```cpp
static chatbot_config_t*  get_config_dialogflow(mesibo_module_t* mod){
        chatbot_config_t* cbc = (chatbot_config_t*)calloc(1, sizeof(chatbot_config_t));
        cbc->project = mesibo_util_getconfig(mod, "project");
        cbc->session = mesibo_util_getconfig(mod, "session");
        cbc->endpoint = mesibo_util_getconfig(mod, "endpoint");
        cbc->access_token = mesibo_util_getconfig(mod, "access_token");
        cbc->chatbot_uid = mesibo_util_getconfig(mod, "chatbot_uid");
        cbc->log = atoi(mesibo_util_getconfig(mod, "log"));

        return cbc;
}
```

### Initialization of REST API PARAMETERS

Once the configuration is obtained, the REST API parameters (URL and header) are constructed so that we can use it when required, rather than constructing them at runtime.


```cpp
static int chatbot_init_dialogflow(mesibo_module_t* mod){
        chatbot_config_t* cbc = (chatbot_config_t*)mod->ctx;
        mesibo_log(mod, cbc->log, "chatbot_init_dialogflow called \n");

        int cv;
        cv = asprintf(&cbc->post_url,"%s/projects/%s/agent/sessions/%s:detectIntent",
                        cbc->endpoint, cbc->project, cbc->session);
        if(cv == -1)return MESIBO_RESULT_FAIL;
        mesibo_log(mod, cbc->log, "Configured post url for HTTP requests: %s \n", cbc->post_url);

        cv = asprintf(&cbc->auth_bearer,"Authorization: Bearer %s", cbc->access_token);
        if(cv == -1)return MESIBO_RESULT_FAIL;
        mesibo_log(mod, cbc->log, "Configured auth bearer for HTTP requests with token: %s \n", cbc->auth_bearer );

        cbc->chatbot_http_opt = mesibo_chatbot_get_http_opt(cbc);

        return MESIBO_RESULT_OK;
}
```

### 3.`chatbot_on_message`

The module only needs to process messages addressed to the configured `address` of the chatbot. All other messages are passed as it is.

```cpp
static mesibo_int_t chatbot_on_message(mesibo_module_t *mod, mesibo_message_params_t *p,
                const char *message, mesibo_uint_t len) {

        chatbot_config_t* cbc = (chatbot_config_t*)mod->ctx;

        if(strcmp(p->to, cbc->chatbot_uid) == 0){
                // Don't modify original as other module will use it
                mesibo_message_params_t* np = (mesibo_message_params_t*)calloc(1, sizeof(mesibo_message_params_t));
                memcpy(np, p, sizeof(mesibo_message_params_t));
                chatbot_process_message(mod, np, message, len);

                return MESIBO_RESULT_CONSUMED;  // Process the message and CONSUME original
        }

        return MESIBO_RESULT_PASS;
}
```
### 4. Processing the query 
 
To process the incoming messages, the module needs to send them to DialogFlow and send the response back to the user.

To invoke Dialogflow API, the helper function `mesibo_http` is called. DialogFlow expects the request data in JSON format. Ideally,  a JSON library could be used to encode the request.  However, JSON libraries are typically slow and are an overkill for this simple project. Hence, we the raw post data string is directly constructed.

Once the response is received from DialogFlow, the module needs to send it to to the user who made the request. Hence, the context of the received message ie; message parameters, the sender of the message, the receiver of the message is stored in the following structure and passed as callback data in the http request. 

```cpp
typedef struct message_context_s {
        mesibo_module_t *mod;
        mesibo_message_params_t *params;
        char *from;
        char *to;
        // To copy data in response
        char buffer[HTTP_BUFFER_LEN];
        int datalen;
     	
char* post_data; //For cleanup after HTTP request is complete
} message_context_t;
```                    

The function to process the message and send an HTTP request to Dialogflow is as follows:

```cpp
static int chatbot_process_message(mesibo_module_t *mod, mesibo_message_params_t *p,
                const char *message, mesibo_uint_t len) {

        chatbot_config_t* cbc = (chatbot_config_t*)mod->ctx;

        const char* post_url = cbc->post_url;
        char* raw_post_data;
        asprintf(&raw_post_data, "{\"queryInput\":{\"text\":{\"text\":\" %.*s \"}}}",
                        (int)len, message);

        message_context_t *message_context =
                (message_context_t *)calloc(1, sizeof(message_context_t));
        message_context->mod = mod;
        message_context->params = p;
        message_context->from = strdup(p->from);
        message_context->to = strdup(p->to);

        mesibo_http(mod, post_url, raw_post_data, chatbot_http_callback,
                        (void *)message_context, cbc->chatbot_http_opt);

        return MESIBO_RESULT_OK;
}
```

### 5. Extracting the response text

The response for the POST request is obtained in the HTTP callback function passed to `mesibo_http`. The response may be recieved in multiple chunks. Hence the response data is stored in a buffer untill the complete response is received. 

Dialogflow sends the response as a JSON string with the response text encoded in the field `fulfillmentText`. Hence, first the response from the JSON string is extracted before  it can be sent back to the user. The helper function `mesibo_util_json_extract` is used to extract textual response from the JSON response.

The message-id of the query message is passed as reference-id for the response message. This way the client who sent the message will be able to match the response received with the query sent. 


```cpp
static int chatbot_http_callback(void *cbdata, mesibo_int_t state,
                mesibo_int_t progress, const char *buffer,
                mesibo_int_t size) {
        message_context_t *b = (message_context_t *)cbdata;
        mesibo_module_t *mod = b->mod;
        chatbot_config_t* cbc = (chatbot_config_t*)mod->ctx;

        mesibo_message_params_t *params = b->params;

        if (progress < 0) {
                mesibo_chatbot_destroy_message_context(b);
                return MESIBO_RESULT_FAIL;
        }

        if (state != MODULE_HTTP_STATE_RESPBODY) {
                return MESIBO_RESULT_OK;
        }

        if ((progress > 0) && (state == MODULE_HTTP_STATE_RESPBODY)) {
                memcpy(b->buffer + b->datalen, buffer, size);
                b->datalen += size;
        }

	if (progress == 100) {

                char* extracted_response = mesibo_util_json_extract(b->buffer , "fulfillmentText", NULL);

		mesibo_message_params_t p;
                memset(&p, 0, sizeof(mesibo_message_params_t));
                p.id = rand();
                p.refid = params->id;
                p.aid = params->aid;
                p.from = params->to;
                p.to = params->from; // User adress who sent the query is the recipient
                p.expiry = 3600;

		mesibo_message(mod, &p, extracted_response , strlen(extracted_response));

                mesibo_chatbot_destroy_message_context(b);
        }

        return MESIBO_RESULT_OK;
}
```

### 6. Compiling the chatbot module
To compile the chatbot module from source run
```
make
```
from the source directory which uses the sample `Makefile` provided to build a shared object `mesibo_mod_chatbot.so`. It places the result at the `TARGET` location `/usr/lib64/mesibo/mesibo_mod_chatbot.so` which you can verify.

### 7. Loading the chatbot module
 
To load the chatbot module provide the configuration in `/etc/mesibo/mesibo.conf`. Refer `sample.conf`.

Mount the directory containing your library which in this case is `/usr/lib64/mesibo/`, while running the mesibo container
as follows. You also need to mount the directory containing the mesibo configuration file which in this case is `/etc/mesibo`

For example, if `mesibo_mod_chatbot.so` is located at `/usr/lib64/mesibo/`
```
sudo docker run  -v /certs:/certs -v  /usr/lib64/mesibo/:/usr/lib64/mesibo/ \
-v /etc/mesibo:/etc/mesibo
-net=host -d  \ 
mesibo/mesibo <app token>
