# Intro

This is the crAPI challenge solutions page. Go through the [challenges page](challenges.md) to get an idea about which vulnerabilities exist in crAPI.

# Challenge Solutions

It is assumed that the container crapi-web is running in port 8888.
Verify the ports of the container by running the following command : `docker ps`

## BOLA Vulnerabilities

### [Challenge 1 - Access details of another user’s vehicle](challenges.md#challenge-1---access-details-of-another-users-vehicle)

#### Detailed solution

1. Login to the application from http://localhost:8888/login
2. From the *Dashboard*, choose *Add a Vehicle* and add the vehicle by providing the VIN and pincode received in Mailhog mailbox after Signup or by reinitiating from *Dashboard* page.
3. After the vehicle details are verified successful, the vehicle will get added and then be populated in the *Dashboard* page.
4. Observe the request sent when we click *Refresh Location*. It can be seen that the endpoint is in the format `/identity/api/v2/vehicle/<vehicleid>/location`.
5. Sensitive information like latitude and longitude are provided back in the response for the endpoint. Send the request to *Repeater* for later purpose.
6. Click *Community* in the navbar to visit http://localhost:8888/forum
7. It can be observed that the forum posts are populated based on the response from `/community/api/v2/community/posts/recent` endpoint. On further analysing the response, it can be seen that `vehicleid` is also received back corresponding to the author of each post.
8. Edit the vehicleid in the request sent to *Repeater* in Step 5 with the `vehicleid` received from endpoint `/community/api/v2/community/posts/recent`.
9. Upon sending the request, sensitive details like latitude, longitude and full name are received in the response.

The above challenge was completed using Burp Suite Community Edition.

### [Challenge 2 - Access mechanic reports of other users](challenges.md#challenge-2---access-mechanic-reports-of-other-users)

#### Detailed solution

1. Login to the application from http://localhost:8888/login
2. After adding a vehicle, we will have an option to send service request to mechanic by using the *Contact Mechanic* option.
3. Observe the request sent after the *Send Service Request*. In the response of `/workshop/api/merchant/contact_mechanic`, we will be able to find the hidden API endpoint in `report_link`.
4. Go to the link present as value in `report_link`.
5. Change the value of report_id in the request and send it to access mechanic reports of other users.


## Broken Authentication 

### Challenge 3 - Reset the password of a different user

#### Detailed solution

1. Navigate to the application’s “forgot password” page.
2. Input the victim’s email address and click “Send OTP.”
3. Now try changing the password with an invalid OTP and capture the request
4. Fuzz the OTP with Intruder and with range from 0000-9999 as the payload.
5. Note that the application has a rate-limit set. Observe the 503 status code and response body.
6. Downgrade versioning from v3 to v2 and repeat the attack. (API9:2023 Improper Inventory Management)
7. Note that you will successfully brute-force the OTP as v2 has no rate-limit set.

### Challenge 14 - Find an endpoint that does not perform authentication checks for a user.

1. Authenticate to the application and Navigate to the “Shop” page.
2. Capture a "Past Orders" request `/workshop/api/shop/orders/all`.
3. Remove Authorization token and send the request.
4. Note that you receive a 401 response.
5. Change the endpoint `/workshop/api/shop/orders/all` to `/workshop/api/shop/orders/<id>` and replace `id` with a valid id.
6. Note that you receive a 200 response containing all details of that order.

## Excessive Data Exposure (API3:2023 Broken Object Property Level Authorization)

### Challenge 4 - Find an API endpoint that leaks sensitive information of other users

#### Detailed solution

1. Authenticate to the application and Navigate to the “Community” page of the application.
2. Click on “+ New Post” and add a post.
3. Now Navigate back to the Forum page and capture the request to `/community/api/v2/community/posts/recent`.
4. Although the application does not display another user’s email, the API response does.

### Challenge 5 - Find an API endpoint that leaks an internal property of a video

#### Detailed solution

1. Authenticate to the application and Navigate to the “Profile” page.
2. Note that a user can upload any personal video to the application.
3. Capture the video upload request. Look at the “coversion_params” parameter in the response, which leaks the conversion codec
Bonus. Capture the request to change the name of the video. Add the “coversion_params” parameter in the body and inject arbitrary commands into the value. Send the request and note that the changes are reflected in the response. (Mass Assignment could escalate to Remote Code Execution)
   
## Rate Limiting (API4:2023 Unrestricted Resource Consumption)

### Challenge 6 - Perform a layer 7 DoS using ‘contact mechanic’ feature

#### Detailed solution
1. Authenticate to the application and Navigate to the “Dashboard” page.
2. Click on "Contact Mechanic" button.
3. Fill in the required fields and capture the POST request `/workshop/api/merchant/contact_mechanic`.
4. There are two parameters exposed "repeat_request_if_failed":false,"number_of_repeats":1.
5. Change those to to cause a DoS attack "repeat_request_if_failed":true,"number_of_repeats":1000.
6. Note in the reponse the message "Service unavailable. Seems like you caused layer 7 DoS :)"


## BFLA 

### Challenge 7 - Delete a video of another user

#### Detailed solution
1. Authenticate to the application and Navigate to the “Profile” page.
2. Click "change video name" and capture the PUT request to `/identity/api/v2/user/videos/<id>`.
3. Replace PUT method with OPTIONS and note that you can use the DELETE method. 
4. Use DELETE on the current `id` and note the 403 response with message "This is an admin function. Try to access the admin API". (BOFLA)
5. Change `user` path in the endpoint `/identity/api/v2/user/videos/<id>` to `admin` and repeat the request.
6. Note the response message "User video deleted successfully".

Bonus
1. Navigate to the “Shop” page.
2. Capture the request of the products page `/workshop/api/shop/products`.
3. Replace GET method with OPTIONS and observe that POST is allowed.
4. Send an empty json using POST and note the required fieds.
5. Create a new product using those fields (price can have negative value) and send the request.
6. A new product is created which should be only permitted by admins.

## Mass Assignment (Broken Object Property Level Authorization)

### [Challenge 8 - Get an item for free](challenges.md#challenge-8---get-an-item-for-free)

#### Detailed solution
1. Authenticate to the application and Navigate to the “Shop” page.
2. There is an initial available balance of $100. Try to order the *Seat* item for $10 from the shop by using the *Buy* button and observe the request sent.
3. On observing the POST request `/workshop/api/shop/orders`, it can be observed that `credit` has been reduced by $10 and the current available balance is $90.
4. With this knowledge, we can try to send the captured POST request `/workshop/api/shop/orders` to *Repeater*. 
5. Try to change the value of `quantity` in the request body to a negative value and send the request. It can be observed that the available balance has now increased and the order has been placed.
6. We can verify that the order has been placed by going to the Past Orders section and thus completing the challenge.

### [Challenge 9 - Increase your balance by $1,000 or more](challenges.md#challenge-9---increase-your-balance-by-1000-or-more)

It is recommended to complete *Challenge 8 - Get an item for free* before attempting this challenge.

#### Detailed solution
1. Login to the application from http://localhost:8888/login
2. Click *Shop* in the navbar to visit http://localhost:8888/shop
3. There is an initial available balance of $100. Try to order the *Seat* item for $10 from the shop by using the *Buy* button and observe the request sent.
4. On observing the POST request `/workshop/api/shop/orders`, it can be observed that `credit` has been reduced by $10 and the current available balance is $90.
5. With this knowledge, we can try to send the captured POST request `/workshop/api/shop/orders` to *Repeater*. 
6. Try to change the value of `quantity` in the request body to a negative value and send the request. It can be observed that the available balance has now increased and the order has been placed.
7. Inorder to increase the balance by $1000 or more, provide an appropriate value in the ‘quantity’ (ie: -100 or less) and send the request. It can be observed that the available balance has now increased by $1000 or more.

Bonus.
1. Navigate to "Past Orders" and return one of the orders.
2. Capture request to `/workshop/api/shop/orders/<id>` and send to Repeater.
3. Observe in the response status "return pending".
4. Change method from GET to PUT and in the body add the json parameter `"status":"returned"` and `"quantity":10`
5. Note that your balance will increase from 100$ to 200$ even if you never purchased 10 items.

### Challenge 10 - Update internal video properties

1. Authenticate to the application and Navigate to the “Profile” page.
2. Capture the “change name to video” request. Add the “coversion_params” parameter in the body of the request (Mass Assignment)
3. Inject arbitrary code into the value. Send the request and note that the changes are reflected in the response.


## SSRF

### Challenge 11 - Make crAPI send an HTTP call to https://www.google.com and return the HTTP response. 

1. Authenticate to the application.
2. Capture the POST request to "contact mechanic" which is `/workshop/api/merchant/contact_mechanic` and send.
3. Note on the response body parameter `report_link` contains url link with a unique `id`.
4. Copy and paste the URL to the browswer, change to a different `id` to disclosure the report of a different user. (Broken Authentication).
5. Now replace the request parameter `mechanic_api` to `https://www.google.com` and send the request.
6. Note the response contains the the source code of google website effectively completing a SSRF.
7. You can also enumerate internal resources `https://localhost:8888/admin` (Not found in this case)


## NoSQL Injection

### Challenge 12 - Find a way to get free coupons without knowing the coupon code.

## SQL Injection

### Challenge 13 - Find a way to redeem a coupon that you have already claimed by modifying the database

## JWT Vulnerabilities

## [Challenge 15 - Find a way to forge valid JWT Tokens](challenges.md#challenge-15---find-a-way-to-forge-valid-jwt-tokens)

#### Detailed Solution

##### crAPI is vulnerable to to the following JWT Vulnerabilities
1. JWT Algorithm Confusion Vulnerability
   - crAPI uses RS256 JWT Algorithm by default
   - Public Key to verify JWT is available at http://localhost:8888/.well-known/jwks.json
   - Convert the public key to base64 encoded form and use it as a secret to create a JWT in HS256 Algorithm
   - This JWT will be accepted as a valid JWT Token by crAPI
2. Invalid Signature Vulnerability
   - User Dashboard API is not validating JWT signature
   - Create a JWT with `sub` header set to a different user's email
   - With the above JWT you will be able to extract user data from user dashboard API endpoint
3. JKU Misuse Vulnerability
   - crAPI will verify JWT token with any public key that is pointed to by the `jku` JWT header
   - Create your own public/private key pair and sign a JWT in RS256 Algorithm
   - Host the public key somewhere in JWK format
   - Pass the public key URL in `jku` header of the JWT with appropriate `kid` header
   - This JWT will be accepted as a valid JWT Token by crAPI
4. KID Path Traversal Vulnerability
   - Set the `kid` header of JWT to `../../../../../../dev/null`
   - Create a custom JWT in HS256 algorithm with secret as `AA==`
     - `AA==` is the Base64 encoded form of Hex null byte `00`
   - This JWT will be accepted as a valid JWT Token by crAPI

## << 2 secret challenges >>
