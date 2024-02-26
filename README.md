# Web-Application-Security-Assessment
<h1>Target: http://testphp.vulnweb.com/ </h1>

<h3>1) Nmap Scan</h3>

 ```
 nmap -v -O -T4 -p- -A testphp.vulnweb.com
  ```
<img alt="" width="700" height="500" src="https://i.ibb.co/Rj1Qh4q/NMAP.png"></img>
<ul>
  <li><p>The scan found port 80/tcp open, running HTTP with Nginx version 1.19.0. Supported HTTP methods include GET, HEAD, and POST.</p></li>
  <li><p>The scan also detected an unknown favicon associated with the website.</p></li>
  <li><p>The device type is an Actiontec MI424WR-GEN3I WAP running DD-WRT v24-sp2 on Linux 2.4.37.</p></li>
</ul>
<h3>2) Nikto Scan</h3>

 ```
 nikto -h http://testphp.vulnweb.com/
  ```
<img alt="" width="700" height="500" src="https://i.ibb.co/PMYNHnb/NIKTO.png"></img>
<ul>
  <li><p><code>X-Powered-By Header Disclosure</code> reveals PHP version and distribution information which could aid attackers in crafting targeted attacks against known vulnerabilities in PHP versions.</p></li>
  <li><p>The <code>anti-clickjacking X-Frame-Options header</code> is not present which allows <code>clickjacking</code> attacks.</p></li>
  <li><p>The <code>X-Content-Type-Options</code> header is not set which could allow the user agent to render the content of the site different to the MIME type, potentially leading to MIME sniffing attacks.</p></li>
  <li><p><code>clientaccesspolicy.xml</code> and <code>crossdomain.xml</code> contains a full wildcard entry.<br> These files are related to cross-domain policy and could potentially allow unintended cross-domain access, which might lead to <code>Cross-Site Scripting (XSS)</code> or <code>Cross-Site Request Forgery (CSRF)</code> attacks.</p></li>
</ul>
<h3>2) OWASP ZAP Scan</h3>
<img alt="" width="700" height="450" src="https://i.ibb.co/VtY5M4z/2024-02-26-02-05-03-kali-VMware-Workstation-17-Player-Non-commercial-use-only.png"></img>

# <h2>Vulnerabilities</h2>
<h2>1) SQL injection</h2>
URLs:<ul><li>http://testphp.vulnweb.com/artists.php?artist=3+AND+9858%3D4325</li><br><li>http://testphp.vulnweb.com/listproducts.php?artist=%28SELECT+%28CASE+WHEN+%287701%3D1956%29+THEN+7701+ELSE+7701*%28SELECT+7701+FROM+INFORMATION_SCHEMA.CHARACTER_SETS%29+END%29%29</li></ul>
<code>Description:</code> The page results were successfully manipulated using various boolean conditions.The parameter value being modified was stripped from the HTML output for the purposes of the comparison.Data was returned for the original parameter.<br><br>
URL:<ul><li>http://testphp.vulnweb.com/secured/newuser.php</li></ul>
<code>Description:</code> The query time is controllable using parameter value which caused the request to take [5,597] milliseconds, when the original unmodified query with value [ZAP] took on average [571.476] milliseconds.<br><br>
URL:<ul><li>http://testphp.vulnweb.com/search.php?test=query%27+UNION+ALL+SELECT+NULL%2CCONCAT%280x3a6d74703a%2C0x42624875466846505145%2C0x3a706a623a%29%2CNULL%23</li></ul>
<code>Description:</code> RDBMS [MySQL] likely, given UNION-specific error message fragment for [3] columns.<br>
<br><code>Mitigation:</code> <ol>
<li>Type check all data on the server side.</li>

<li>If database Stored Procedures can be used, use them.</li>

<li>Do *not* concatenate strings into queries in the stored procedure, or use 'exec', 'exec immediate', or equivalent functionality!</li>
<li>Do not create dynamic SQL queries using simple string concatenation.</li>

<li>Escape all data received from the client.</li>

<li>Apply an 'allow list' of allowed characters, or a 'deny list' of disallowed characters in user input.</li>

<li>Grant the minimum database access that is necessary for the application.</li></ol>

<br>
<h2>2) Cross-Site Request Forgery (CSRF)</h2>

<code>Cause:</code> <ul>
  <li>Due to <code>crossdomain.xml</code> misconfiguration on the web server.</li>
  <li><code>Evidence:</code> allow-access-from domain="*"</li></ul>
<code>Description:</code> The web server permits malicious cross-domain data read requests originating from Flash/Silverlight components served from any third party domain, to this domain. If the victim user is logged into this service, the malicious read requests are processed using the privileges of the victim, and can result in data from this service being compromised by an unauthorised third party web site, via the victims web browser.<br><br>
<code>Mitigation:</code> Configure the crossdomain.xml file to restrict the list of domains that are allowed to make cross-domain read requests to this web server, using <code> allow-access-from domain="example.com"</code>.<br><br>
<code>Cause:</code> No Anti-CSRF tokens were found in a HTML submission form [Form 1: "goButton" "searchFor"]<br><br>
<code>Mitigation:</code> <ol>
  <li>Use anti-CSRF packages such as the OWASP CSRFGuard.</li>
  <li>Ensure that your application is free of cross-site scripting issues, because most CSRF defenses can be bypassed using attacker-controlled script.</li>
  <li>Generate a unique nonce for each form, place the nonce into the form, and verify the nonce upon receipt of the form. Be sure that the nonce is not predictable.</li>
  <li>Use the ESAPI Session Management control.</li>
  <li>Do not use the GET method for any request that triggers a state change.</li>
</ol>


<br>
<h2>3) Source Code Disclosure - File Inclusion</h2>
URL:<ul><li>http://testphp.vulnweb.com/hpp/params.php?p=valid&pp=12</li></ul>
<br>
<code>Description:</code> The Path Traversal attack technique allows an attacker access to files, directories, and commands that potentially reside outside the web document root directory. An attacker may manipulate a URL in such a way that the web site will execute or reveal the contents of arbitrary files anywhere on the web server. <br><br>

<code>Mitigation:</code> <ol>
  <li> Say NO to bad data! Use strict allow lists for inputs (length, type, values).</li>
  <li> For filenames: limited characters, single ".", no directory separators, and allow list of extensions.</li>
  <li> Prepare data: decode, convert, use built-in functions for paths.</li>
  <li> Run code with least privilege and consider isolated accounts.</li>
  <li> Map to known values (IDs) when possible.</li>
  <li> Sandbox the code for extra protection.</li>
</ol>


<br>
<h2>4) .htaccess Information Leak</h2>
URL:<ul><li>http://testphp.vulnweb.com/Mod_Rewrite_Shop/.htaccess</li></ul>
<br>
<code>Description:</code> htaccess files can be used to alter the configuration of the Apache Web Server software to enable/disable additional functionality and features that the Apache Web Server software has to offer. <br><br>
<code>Mitigation:</code> <ul>
  <li> Ensure the .htaccess file is not accessible.</li>
</ul>


<br>
<h2>5) XSLT Injection</h2>
<code>Description:</code> Injection using XSL transformations may be possible, and may allow an attacker to read system information, read and write files, or execute arbitrary code.<br><br>
<code>Mitigation:</code> <ul>
  <li> Sanitize and analyze every user input coming from any client-side.</li>
</ul>


<br>
<h2>6) XSS Attack</h2>
<code>Evidence:</code> </li> scrIpt>alert(1);</scRipt><li>
<br>
<code>Description:</code> htaccess files can be used to alter the configuration of the Apache Web Server software to enable/disable additional functionality and features that the Apache Web Server software has to offer. <br><br>
<code>Mitigation:</code> <ul>
  <li> Use trusted libraries for automatic encoding. Know how data moves and use proper encoding throughout. Double-check client-side security on the server. Protect session cookies with HttpOnly flag. Validate all input strictly, assuming it's malicious. Combine these tips for maximum XSS security!</li>
</ul>

<br>
<h2>7) Backup File Disclosure</h2>
<code>Description:</code> A backup of the file was disclosed by the web server <br><br>
<code>Mitigation:</code> <ul>
  <li> Do not edit files in-situ on the web server, and ensure that un-necessary files (including hidden files) are removed from the web server.</li>
</ul>


<br>
<h2>8) Clickjacking</h2>
<code>Description:</code> The server response does not include either Content-Security-Policy with 'frame-ancestors' directive or X-Frame-Options to protect against 'ClickJacking' attacks. <br><br>
<code>Mitigation:</code> <ul>
  <li>  Ensure one of them is set on all web pages returned by your site/app.</li>
</ul>


<br>
<h2>9) Server Information Leak</h2>
<code>Cause:</code> <ul>
  <li> Via "Server" HTTP Response Header Field</li>
  <li> Via "X-Powered-By" HTTP Response Header Field(s)</li></ul>
<code>Description:</code> The web/application server is leaking information via these HTTP response headers. Access to such information may facilitate attackers identifying other frameworks/components your web application is reliant upon and the vulnerabilities such components may be subject to.<br><br>
<code>Mitigation:</code> <ul>
  <li> Ensure that your web server, application server, load balancer, etc. is configured to suppress these headers.</li>
</ul>


<br>
<h2>10) X-Content-Type-Options Header Missing</h2>
<code>Description:</code> The Anti-MIME-Sniffing header X-Content-Type-Options was not set to 'nosniff'. This allows older versions of Internet Explorer and Chrome to perform MIME-sniffing on the response body, potentially causing the response body to be interpreted and displayed as a content type other than the declared content type.<br><br>
<code>Mitigation:</code> <ul>
  <li> Ensure that the application/web server sets the Content-Type header appropriately, and that it sets the X-Content-Type-Options header to 'nosniff' for all web pages.</li>
</ul>








