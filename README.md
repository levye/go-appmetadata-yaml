# go-appmetadata-yaml
![alt text](https://github.com/matryer/gophers/blob/master/GOPHER_AVATARS.jpg)  

https://github.com/levye/go-appmetadata-yaml/blob/master/README.md

REST API to support read/write application metadata as yaml/json payloads with a integrated in-mem db 

- Worker Thread-Pools request processing with channels
- Async Logger using Channels
- Custom validation function integration to handler
- GET and POST to create and get application metadata
- yaml and json payload support
- Acecept header support (application/json) default is yaml
- Logger and Validator middlewares

Project structure:

- ## cmd/

	This is where main function is defined and all the components
	are being utilized and integrated. main func simple performs;
	
	- create async logger
	- create in-memory storage for application metadata
	- create application contex to access common functionality across different modules
	- initialize work queue and dispatcher
	- create server which also initializes handlers
	- listen and serve
	
	(to see how these steps are being implemented, continue reading)
	Sample usage of the server is:

	
	
- ## pkg

	- ###### /logger
	
	Async logger implementation on top of go logger utilizing Go channels.
	It provides three level of logging which is 
	
		-Warning (stdout)

		-Info (stdout)

		-Error (stderr)
		
		-Fatal (stderr) 
		
	For each level, a log object and a channel 
	
	Sample usage for logger is :
	```go
	asyncLogger := logger.CreateAsyncLogger()
	asyncLogger.Log(logger.INFO, "log message..")
	asyncLogger.Log(logger.WARNING, "log message..")
	asyncLogger.Log(logger.ERROR, "log message..")
	asyncLogger.Log(logger.FATAL, "log message..")
	```
	- ###### /memstore
	
	It is a simple in-memory strorage to store application metadata
	Supports Insert and Read methods.
	```go
	type Database interface {

	Insert(key string, val interface{})
	Read(key string) interface{}
	ReadWithParams(params map[string][]string) []interface{}
	}
	```
	
	- ###### /model
	
	This is where we define our Application metadata model. 
	There is no business logic there but only data model itself.
	
	- ###### /server
	
	This is where actual server implementation exists. We use chaning logic for handler by implementing Adapter (Decorator) pattern.
	So that we can add dynamic behaviour without code replication to our handler. In order to do that, we use a middleware signature  
	which looks like:  
	
	```go
	type middleware func(h http.HandlerFunc) http.HandlerFunc
	```
	
	Bacically, we are getting a **HandlerFunc**, we are adding extra behaviour it and returning a new **HandlerFunc**  
	which enables us to be able to chain multiple HandlerFunc. 
	
	Let see one of the example about how to use it:  
	
	```go
	func (s *Server) withLog() middleware {

	s.Log.LogInfo("withLog called")
	return func(h http.HandlerFunc) http.HandlerFunc {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			s.Log.LogInfo("Log Before")
			defer s.Log.LogInfo("Log After")
			h(w, r)
		})
	}
	}
	```
	
	As you can see, we are adding logging capability to our HandlerFunc and inside it, we are calling next HandlerFunc in chain.  
	If you think about the usage of this kind of chaining mechanishm:  
	- You can add logging capabilitiy  
	- You can perform validation  
	- You can decide to call or not to call next handlerFunc depending on your logic  
	- ...  
	
	It is also practicle to chain all your middleware function so code will be more readable.   
	```go
	func (s *Server) Chain(h http.HandlerFunc, m ...middleware) http.HandlerFunc
	```  
	
	So how we acctually use them, see example below,
	
	```go
	type middleware func(h http.HandlerFunc) http.HandlerFunc  
	func (s *Server) createAppMetadataHandler(w http.ResponseWriter, r *http.Request)  
	func (s *Server) withValidation(validator validatorFunc) middleware  
	
	func (s *Server) searchAppMetadataHandler(w http.ResponseWriter, r *http.Request) //actual handler

	s.Routers.HandleFunc("/api/v1/apps", s.Chain(s.createAppMetadataHandler,
		s.withValidation(validator.ValidateRequest),
		s.withLog())).Methods("POST")
	```
	
	The important point here how we chain and use them,  
	
	```go
	s.Chain(s.createAppMetadataHandler,
 		s.withValidation(validator.ValidateRequest),
 		s.withLog())).Methods("POST")
	```
	
	This is a cleaer code to read and implement. So whatever behavior you would like to add to your handler,   
	you can implement it as middleware and chain with other middleware functions.  
	
	
	- ###### /validator
	
	This is a custom validation function to use for handler where we validate incoming request.   
	We have a predefined signature for validator functions so our validate func should have   
	exactly same signature in order to use them. The signature is : 
	
	```go
	type validatorFunc func(h *http.Request) (bool, string)  
	```
	
	Where it takes request as argumant, which we will validate, and return bool and string where bool indicates that  
	if validation is successfull or not, and error string that shows where our validation has failed. (error message)  
	
	


Sample usage of the server is:

```go
	//create async logger
	asyncLogger := logger.CreateAsyncLogger()

	storage := memstore.CreateInMemDB()
	storage.SetLogger(asyncLogger)

	//create application context
	appContext := context.AppContext{
		Storage: storage,
		Logger:  asyncLogger,
	}

	//initialize dispatcher and pools
	workQueue := make(chan workpool.WorkRequest, MaxQueue)
	dispatcher := workpool.NewDispatcher(workQueue, MaxWorker, &appContext)
	dispatcher.StartDispatcher()

	//create server
	server := server.CreateServer(&appContext, workQueue)

	http.Handle("/", server.Routers)
	asyncLogger.Log(logger.INFO, "Listening localhost 8080...")
	err := http.ListenAndServe("localhost:8080", server.Routers)
	if err != nil {
		asyncLogger.Log(logger.FATAL, err.Error())
	}
	```
Sample application metadata payload is :

```yaml
title: My valid app
version: 1.0.8
company: Ecaglar Inc.
website: https://ecaglar.net
source: https://github.com/levye/repo
license: Apache-2.1
maintainers:
  - name: Firstname Lastname
    email: emre@hotmail.com
description: |
    ### blob of markdown
    More markdown
```

## API Details

Server provides a simple enpoint for GET and POST operations.  
**/api/v1/apps**

**POST**  operation is used to create application metadata.   
Yaml or json payload is supported as payload and content type is "application/json"

**GET**  operation is used to read application metadata records. Search parameters are being passed via URL.
Supported URL query search parameters are:  

- { version, title, company, website, source, license, maintainers.name, maintainers.email and description}  
**Note that** title and description parameters are used to check if record **contains!** those parameters.
So full string check does not happen.

## POST OPERATION  

Creates a application metadata. Accepts **yaml** or **json** payload. Both formats are supported. Since yaml is a superset of  
json, yaml parser can also handle json. All fields and valid email addresses are required otherwise returns error. 

In order to optimize workload of server and decrease latency, work thread-pool paradigm has been implemented which is natively 
supported by Go thanks to goroutines, channels and overall native concurrency support of the language. 

Summery of the approach is:

![alt text](https://github.com/levye/image-repo/blob/master/diagram.png)  


Sample POST request is:

```
http://localhost:8080/api/v1/apps  
```

and body payload is:  

```yaml
title: My valid app
version: 1.0.8
company: Ecaglar Inc.
website: https://ecaglar.net
source: https://github.com/levye/repo
license: Apache-2.1
maintainers:
  - name: Firstname Lastname
    email: emre@hotmail.com
description: |
    ### blob of markdown
    More markdown
```

## GET OPERATION  

GET operation also has same endpoint. Changing the URL query parameters, you can query different records.

###### Samples

**POST - /api/v1/apps**  
Accepts application metadata payload in yaml or json format in body.(application/son)

**GET - /api/v1/apps**  
Returns all records

**GET - /api/v1/apps?version=1.0.0**  
Returns the record with version 1.0.0

**GET - /api/v1/apps?version=1.0.0&title=my%20app**  
Returns the record with version 1.0.0 if exists.Does not check other parameters as version number is unique.

**GET - /api/v1/apps?company=mycompany.com&title=my%20app**  
Returns record(s) with company name "mycompany.com" and title **contains** "my app"  

**GET - /api/v1/apps?description=latest**  
Returns record(s) with description **contains** "latest"   

**GET - /api/v1/apps?maintainers.name=Bill&maintainers.name=Joe**  
Returns record(s) which have/has maintainers name "Bill" and "Joe"   

**GET - /api/v1/apps?maintainers.email=bill@c.com&license=Apache-2.1**  
Returns record(s) which have/has maintainers email "bill@hotmail.com" with licence "Apache-2.1"

**TO GET PDF VERSION OF README.md**  

![alt text](https://cloudconvert.com/converter/qrcode?url=//hollie.infra.cloudconvert.com/download/~30c6d053d1b477fb46b5a4c65005556d216f7fb2bb5cb8a554826e72ba0f95d41204facb444074f23f52af00)  


