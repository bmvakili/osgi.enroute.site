---
title: osgi.enroute.rest.api
layout: service
version: 1.0
summary: Provides REST end-points that are based on method naming pattern with type safe use of the pay-load, parameters, and result body.
---

## Introduction

The OSGi enRoute REST API specifies a service contract for components to provide REST _end-points_. Representational State Transfer (REST) is an architectural style that allows the interchange of information between elements of a distributed system. A REST end-point has an HTTP(S) URL and can thus be accessed from all modern computing environments. An end-point defines the meaning of the segments of this URL, any specified parameters in the URL, as well as the used HTTP verb (GET, PUT, POST, etc). For example:

	GET /rest/upper/<word>?alphaonly=true

This end-point is mapped to the URI `/rest/upper` and the next segment specifies the word to translate to upper case. The `alphaonly` is a _parameter_ on the URL, in this case a boolean.

A HTTP request can specify a _payload_, this is normally associated with the `POST` and the `PUT` verb. As response, the HTTP request will return a _body_.

Since the REST API is of such major importance for many modern systems, it is crucial that the overhead for the programmer to support this interface is absolutely minimized. This specifications leverages the Java type system to provide REST end points. It provides a deterministic mapping from a URI request to a REST method name of a restricted set of methods. Adding a method named according to a defined pattern is all that is required to add a new REST end-point.

## REST Methods

The first parameter of a REST method must be an interface that extends the `RESTRequest` interface. This interface provides access to the underlying servlet objects as well as the host name. However, these objects are rarely needed; the primary purpose of this interface is to be extended with methods that have the name of the URL arguments. The interface does not have to be public, it can be a private interface of a class. That is, the previous example would need an interface defined as:

	interface UpperRequest extends RESTRequest {
		boolean alphaonly();
	}  

Any return type can be used for the REST methods that is supported by the DTO conversion techniques. Note that the actual return type must be a public class and have a public constructor.

Any remaining segments in the URI are mapped to the parameters of the method. These method parameters are not required to be strings. Any parameter type that can be converted from a string according to the DTO conversion techniques can be used. The REST method can either have a fixed number of parameters or it can use varargs for a variable number.

The returned body is defined by the method's return type. In general, this type is converted to a JSON file according to the DTO JSON conversion rules. All Java's basic types and all DTO's can be used as return types.

Since REST requests are always copied (they have to move to another process) it is allowed to return original copies; the REST implementation must not modify the returned objects in any way.

Therefore, the previous example can be defined as:

	@Component
	public class UpperApplication implements REST {
	
		interface UpperRequest extends RESTRequest {
			boolean alphaonly();
		}
		  
		public String getUpper( UpperRequest request, String string ) {
			return string.toUpperCase();
		}
	}
	
Assuming a default root of `/rest`, this will provide a REST end-point of the earlier example URL.

## Extra Conversions for the Body

Certain return types of the REST method are not mapped to JSON but are treated differently. These are the special conversions: 

* `InputStream` – Will be copied directly to the output.
* `File` – The File contents will be copied to the output.
* `byte[]` – The content of the byte array will be copied to the output
* `null` – Nothing will be copied to the output. The method could have used the servlet objects to send output for rare cases.
 
If the content type has not been set by the method then the default MIME type will be `application/octet-stream` for these conversions.

## Pay-loads

`POST` and `PUT` URLs carry a pay-load from the client to the server. For this specification this pay-load must be expressed as a JSON body (`application/json;charset=UTF-8`). No other type of bodies are allowed.

The Java type of the pay-load is defined by the return type of the `_body()` method on the request parameter of the REST method. The incoming JSON pay-load is mapped to this return type following the JSON DTO conversion rules. 

For example, a system has to handle people, so there is a (we know, simplistic) Person record.

	public class Person extends DTO {
		publci String id;
		public String name;
		public String middle;
		public String surname;
		public int birthYear;
	}

In REST protocols, the `PUT` verb would be used to store a new person. To create the proper end-point, we can define the following REST method.

	interface PersonRequest extends RESTRequest {
		Person _body();
	}
	
	Person putPerson( PersonRequest request ) {
	
		// authorization
		
		Person p = request._body();
		
		// augment
		// validate
		// persist, set id
		
		return p;
	}
	
## Exceptions

Since the REST methods provide full type safe access to the parameters and remaining URL segments a significant amount of validation is executed by the implementation of this service. These validations will result in the appropriate HTTP error and status code. Implementation should also add explanatory texts to the response.

Additionally, the REST methods may throw any exception, if an exception is thrown it is also translated to an HTTP error code. The conversions from exception to status code is as follows:

* File Not Found Exception – 404 NOT FOUND
* Security Exception – 403 FORBIDDEN

All other exceptions are translated to a 500 SERVER ERROR error code.

Clients can always set the response of the request through the servlet objects that are available on the RESTRequest arguments. However, this should in general be a last resort since most incompatibilities are caused by the sometimes really subtle interpretations of these error codes. In general it is best to try to make requests binary: succeed when all goes OK and fail in all other cases.


## Configuration

Implementations must follow the PID `osgi.enroute.rest` which must support at least the following fields:

* `alias` – The primary end-point
