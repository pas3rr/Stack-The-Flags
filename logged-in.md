# Stack-The-Flags
## Writeup for the web category - logged in 

## DESCRIPTION
It looks like COViD's mobile application is connecting to this API! Fortunately, our agents stole part of the source code. Can you find a way to log in?

## COMMENT
As this looks like a challenge for source code review, we will try to adapt some of the OSWE methodology in reviewing the application flaws. 

## Skill Required 
JS basic syntax
API endpoint 

---

```
apt-get install tree
```

```
tree -L 3 
```

We will start with understanding the basic structure of the folder at basic level, this give us a idea of how's the folder structure and 
where some of the key files are located. 

![image](https://user-images.githubusercontent.com/32186957/101722233-bfd08380-3ae4-11eb-9a96-203aa0dabb55.png)


```
grep -rn "api" .
```

We know that the vector is api, lets check where the api are located, the "app.js" file looks promising

![image](https://user-images.githubusercontent.com/32186957/101722550-716fb480-3ae5-11eb-90f3-62c77d62ffa3.png)

```
subl app.js
```

I prefer sublime for simple editing, you can use any other command like gedit.  Here we can see that 
for app.js to work, we would need the reoutes/api.js file. require in js works like import in python

![image](https://user-images.githubusercontent.com/32186957/101722708-c7445c80-3ae5-11eb-9186-8bb1ce642f95.png)

```
subl ./routes/api.js
```

Here the very interesting things we can see for the api call is the router.get is equivalent to a get request function
on the path /user/:userId, next the async function takes in two parameter req and res (short for request and response)

![image](https://user-images.githubusercontent.com/32186957/101722737-d62b0f00-3ae5-11eb-85cc-a27dac1380be.png)

it will assign the variable user by calling the User.findByPK function which takes in another two parameters 
1. req.params.userId
2. { "attributes": ["username"] } which is a dictionary pair with the keypair "username"

and return

res.json(user), which is the user 

 at this point of time we can use burp to see if we make this API call, however the colon itself means something within JS express, it its actually an URL parameters
https://stackoverflow.com/questions/32313553/what-does-a-colon-mean-on-a-directory-in-node-js

![image](https://user-images.githubusercontent.com/32186957/101723038-6ec18f00-3ae6-11eb-89a9-8f746068db72.png)

so that it will roughly translate to http://xxx/user/1 if the user id is 1

however we still need one more level of directory shown by the code earlier in app.js
which translate to /api/user/:userId

![image](https://user-images.githubusercontent.com/32186957/101723071-8436b900-3ae6-11eb-8d3f-09e8feddba12.png)

This is the http get request for the correct call
```
http://yhi8bpzolrog3yw17fe0wlwrnwllnhic.alttablabs.sg:41061/api/user/1
```
Rather disappointing that it shows as null, however we can confirm that we are on the right track, if it's not showing not found. So lets try a few more value. 

![image](https://user-images.githubusercontent.com/32186957/101723397-2b1b5500-3ae7-11eb-9c01-1e8598bb2ca7.png)

Success !

![image](https://user-images.githubusercontent.com/32186957/101723444-3ec6bb80-3ae7-11eb-9792-aec5d5ed4c02.png)


Lets dig around the source code some more

```
subl ./routes/api.js
```

For the /login api call we can see that it takes in a couple of important function that need to be investigated. Let try to investigate loginValidator function

![image](https://user-images.githubusercontent.com/32186957/101723474-5140f500-3ae7-11eb-91c9-76ec3e51b9d8.png)

```
grep -rn "loginValidator" .
```
![image](https://user-images.githubusercontent.com/32186957/101723496-5d2cb700-3ae7-11eb-86b4-6496ea6ade58.png)

```
subl ./middleswares/validators.js
```

From the validators.js file we can see that the function loginValidator checks for two parameters here. 
- username
- password

![image](https://user-images.githubusercontent.com/32186957/101723537-6d449680-3ae7-11eb-93f3-c76cd88d6939.png)


This information is interesting for us to craft a burp request to see if we are on the correct vectors
/api/login with a post request will be the URL to call the api

```
POST 
http://yhi8bpzolrog3yw17fe0wlwrnwllnhic.alttablabs.sg:41061/api/login
```

We have a 401 unauthorized which shows that we are in the correct vectors here, is there a way to bypass the checks? 

![image](https://user-images.githubusercontent.com/32186957/101723779-f1971980-3ae7-11eb-93a3-08ad06ab6275.png)

Let try with some simple fuzzing,  empty payload 

![image](https://user-images.githubusercontent.com/32186957/101723798-ffe53580-3ae7-11eb-8968-58690a9de36f.png)

Didn't expect to get it success on one try. Lets check the /api/login and see what is happening underneath

![image](https://user-images.githubusercontent.com/32186957/101723815-0a073400-3ae8-11eb-941c-13ed0f587270.png)



loginValidator function checks for the existence of the username and password parameters which we already knows, 

sendValidationErrors checks for if there is any errors and response with the error  object with a 400. Which we can see in burp if we 
try to use a invalid parameters such as just username without password. 

![image](https://user-images.githubusercontent.com/32186957/101723855-24411200-3ae8-11eb-9a90-f4917f125d33.png)

localAuthenticator function is the last which we try to understand, basically it let you  authenticate using a username and password in your Node.js applications
http://www.passportjs.org/packages/passport-local/

![image](https://user-images.githubusercontent.com/32186957/101723886-3753e200-3ae8-11eb-8554-aa7ed219fdaa.png)


Here we understand that the logic is that if there is any err, it will return a 401 not authorized message which we see earlier. So right here at the end of all the 
passed in function, we dont see any sort of validation for the data type of the parameter passed. This is likely the reason why we get that flag in the first place.

![image](https://user-images.githubusercontent.com/32186957/101723926-4dfa3900-3ae8-11eb-930b-4f5d1f760a36.png)
![image](https://user-images.githubusercontent.com/32186957/101723947-58b4ce00-3ae8-11eb-8d30-5075c90d7300.png)



This is a pretty fun one and helps me to better prepare for my OSWE exam! 



