# Configuring Nifi with Ldap User Sync for Authentication

## What is Ldap?
LDAP stands for Lightweight Directory Access Protocol. One of the uses is to store your credentials in a network security system and retrieve it with your password and decrypted key giving you access to the services.
 
Users can integrate LDAP with NiFi to enable Username/password Login authentication from the Nifi UI.
 
In this Stage we will look into the configuration settings and how to Map the LdapSearch output to the login-identity providers settings 


##Configuration pre-requisites:
 
	- NFI Secured via SSL 
	- For Nifi LDAP Authentication to be  configured, SSL must be enabled on the cluster.
	- Check page for SSL setup for Nifi

##Configuration Validation

   1. Check property is set to 
	
     nifi.security.user.login.identity.provider=ldap-provider
 
   2. Configure Ldap info in login-identity-providers setting
        
	 <provider>
         <identifier>ldap-provider</identifier>
         <class>org.apache.nifi.ldap.LdapProvider</class>
         <property name="Authentication Strategy">SIMPLE</property>
         <property name="Manager DN"></property>
         <property name="Manager Password"></property>
         <property name="TLS - Keystore"></property>
         <property name="TLS - Keystore Password"></property>
         <property name="TLS - Keystore Type"></property>
         <property name="TLS - Truststore"></property>
         <property name="TLS - Truststore Password"></property>
         <property name="TLS - Truststore Type"></property>
         <property name="TLS - Client Auth"></property>
         <property name="TLS - Protocol"></property>
         <property name="TLS - Shutdown Gracefully"></property>
         <property name="Referral Strategy">FOLLOW</property>
         <property name="Connect Timeout">10 secs</property>
         <property name="Read Timeout">10 secs</property>
         <property name="Url"></property>
         <property name="User Search Base"></property>
         <property name="User Search Filter"></property>
         <property name="Identity Strategy">USE_DN</property>
         <property name="Authentication Expiration">12 hours</property>
     </provider>

 
 | Authentication Expiration | -- The duration of how long the user authentication is valid for. If the user never logs out, they will be required to log back in following this duration.
| Authentication Strategy | -- How the connection to the LDAP server is authenticated. Possible values are ANONYMOUS, SIMPLE, LDAPS, or START_TLS.
| Manager DN | -- The DN of the manager that is used to bind to the LDAP server to search for users.
| Manager Password | -- The password of the manager that is used to bind to the LDAP server to search for users.
| User Search Base | -- Base DN for searching for users (i.e. CN=Users,DC=example,DC=com).
 | User Search Filter | -- Filter for searching for users against the 'User Search Base'. (i.e. sAMAccountName={0}). The user specified name is inserted into '{0}'.
| Identity Strategy | -- Strategy to identify users. Possible values are USE_DN and USE_USERNAME. The default functionality if this property is missing is USE_DN in order to retain backward compatibility. USE_DN will use the full DN of the user entry if possible. USE_USERNAME will use the username the user logged in with.
 
The Rest of the TLS parameters are mentioned when connecting to LDAP via LDAPS or START_TLS.  i.e , Authentication Strategy is set to LDAPS or START_TLS in the login-identity-providers setting file.
 
 
Sample :
	```markd
	<provider>        
	<identifier>ldap-provider</identifier>        
	<class>org.apache.nifi.ldap.LdapProvider</class>       
	<property name="Authentication Strategy">SIMPLE</property>         
	<property name="Manager DN">cn=admin,dc=example,dc=com</property>        
	<property name="Manager Password">password</property>         
	<property name="Referral Strategy">FOLLOW</property>        
	<property name="Connect Timeout">10 secs</property>        
	<property name="Read Timeout">10 secs</property>         
	<property name="Url">ldap://ldaphost:389</property>        
	<property name="User Search Base">dc=example,dc=com</property>        
	<property name="User Search Filter">(uid={0})</property>         
	<property name="Identity Strategy">USE_USERNAME</property>         
	<property name="Authentication Expiration">12 hours</property>    
	</provider>
	```

LDAP Search Output :

Command :
> ldapsearch  -h 000.11.111.00 -p 389 -D "cn=manager,dc=charan,dc=com" -w password -b dc=charan,dc=com  

Output :
# sai, People, charan.com
dn: uid=sai,ou=People,dc=charan,dc=com
shadowWarning: 0
uid: sai
uidNumber: 10005
gecos: sai
shadowLastChange: 0
homeDirectory: /home/sai
gidNumber: 500
shadowMax: 0
cn: sai
objectClass: shadowAccount
objectClass: posixAccount
objectClass: account
objectClass: toplogin
Shell: /bin/bash
userPassword:: YWRtaW4= 
 

##How to map the objects : 
 
###Manager DN:
It is the Bind DN of the ldap, which needs to be obtained from the ldap Configuration and it is the one with which you can run the ldapserach command.
 
###User Search Base:
In the baseDN of the ldap hierarchy under which the user will exists . You can relate this from the -b option of you ldapsearch command 
 
###User Search Filter:
This is to options present to determine is the user is present within the baseDN .
For Example : we have the baseDN as "dc=charan,dc=com", there are can N # of users, and to filter just the users, we use the UID attribute as the filter option, to identity the users., which is set as "uid={0}" in the configuration and it equals to "uid: sai", when user "sai" is logged-in
 
When you AD with ldap, you will mostly come across "sAMAcountName" as the user search filter.
 
 
 
If the Customer needs to have the User search based on the a LDAP group, then the "User Search Filter" property can be set as below. 
In the below example, the User will be searched in LDAP group1 and group2 . You can assign required permission via Ranger or File-Based Authorizations for the groups, which will be in effect for the users that belong to the group.
 
###Example :
```mark
<property name="User Search Filter">(|(memberOf=CN=group1,OU=Groups,OU=BigData,DC=EXAMPLE,DC=COM)(memberOf=CN=group2,OU=Groups,OU=BigData,DC=EXAMPLE,DC=COM))</property>
 ```
 
##Files Affected :
	1. nifi.properties 
	2. login-identity-providers.xml
	3. users.xml 
 
Troubleshooting Info needed :
	1. Config Files : nifi.properties , login-identity-providers.xml, users.xml 
	2. Ldapsearch command output
	3. nifi-users.log and the nifi-app.log
