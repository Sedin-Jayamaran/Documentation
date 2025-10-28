# HTTP STATUS 
## STATUS CODE :
### There are many status code like 2XX , 3XX , 4XX , 5XX family . They show us the end result of the request made .
#### 2XX Family :
- 200 - Request is successful
- 201 - Success , but something is created . eg , If I create a branch on github through API call , It create a branch or resource.
- 204 - A special case of 200 , used when there is no content in the server to return to the client. eg , Generally used in deletion request .
  
#### 3XX Family :
- 300 - Success , but redirection is involved. eg , youtube redirection . 
- 301 - Redirected permanently.
- 302 - Redirected temporarly.

#### 4XX Family : A red flag status code 

- 400 - Bad Request . ie , Your request itself has some issue .
- 401 - Unauthorized or Unauthenticated , Your api call has been denied .
- 403 - Forbidden , a particular section of api is blocked . Like you are not authorized to access the session .
- 404 - Not Found error . 
  
#### 5XX Family :  Problem in server

- 500 - Internal server error . Means there is a issue with the server in which the application is deployed . Example like scalability issue , time out , etc ..
- 502 - Bad Gateway . I am depended on a specific service which is not working properly . eg , if your frontend , backend responds well and database fails , it results in BAD GATEWAY.