DESCRIPTION
It looks like COViD's mobile application is connecting to this API! Fortunately, our agents stole part of the source code. Can you find a way to log in?

COMMENT
As this looks like a challenge for source code review, we will try to adapt some of the OSWE methodology in reviewing the application flaws. 

Skill Required 
JS basic syntax
 API endpoint 

---
Enumeration

```
apt-get install tree
```

```
tree -L 3 
```

We will start with understanding the basic structure of the folder at basic level, this give us a idea of how's the folder structure and 
where some of the key files are located. 


```
grep -rn "api" .
```

We know that the vector is api, lets check where the api are located, the "app.js" file looks promising


```
subl app.js
```

I prefer sublime for simple editing, you can use any other command like gedit.  Here we can see that 
for app.js to work, we would need the reoutes/api.js file. require in js works like import in python


```
subl ./routes/api.js
```

Here the very interesting things we can see for the api call is the router.get is equivalent to a get request function
for the path /user/:userId which the async function takes in two parameter req and res (short for request and response)

it will assign the variable user by calling the User.findByPK function which takes in another two parameters 


and return

 at this point of time we can use burp to see if we make this API call, however the colon itself means something within JS express, it its actually an URL parameters
https://stackoverflow.com/questions/32313553/what-does-a-colon-mean-on-a-directory-in-node-js

so that it will roughly translate  to http://xxx/user/1 if the user id is 1

```
http://yhi8bpzolrog3yw17fe0wlwrnwllnhic.alttablabs.sg:41061/api/user/1
```

however we still need one more level of directory shown by the code earlier in app.js
which translate to /api/user/:userId

Rather disappointing that it shows as null, however we can confirm that we are on the right track, if it's not showing not found. So lets try a few more value. 

Success !


Lets dig around the source code some more

```
subl ./routes/api.js
```

For the /login api call we can see that it takes in a couple of important function that need to be investigated. Let try with  validators.js


```
grep -rn "loginValidator" .
```



```
subl ./middleswares/validators.js
```

From the validators.js file we can see that the function loginValidator checks for two parameters here. 
- username
- password

This information is interesting for us to craft a burp request to see if we are on the correct vectors
/api/login with a post request will be the URL to call the api

```
POST 
http://yhi8bpzolrog3yw17fe0wlwrnwllnhic.alttablabs.sg:41061/api/login
```

We have a 401 unauthorized which shows that we are in the correct vectors here, is there a way to bypass the checks? 

Let try with some simple fuzzing,  empty payload 

Didn't expect to get it success on one try. Lets check the /api/login and see what is happening underneath


loginValidator function checks for the existence of the username and password parameters which we already knows, 

sendValidationErrors checks for if there is any errors and response with the error  object with a 400. Which we can see in burp if we 
try to use a invalid parameters such as just username without password. 



localAuthenticator function is the last which we try to understand, basically it let you  authenticate using a username and password in your Node.js applications
http://www.passportjs.org/packages/passport-local/

Here we understand that the logic is that if there is any err, it will return a 401 not authorized message which we see earlier. So right here at the end of all the 
passed in function, we dont see any sort of validation for the data type of the parameter passed. This is likely the reason why we get that flag in the first place.
This is a pretty fun one and helps me to better prepare for my OSWE exam! 



