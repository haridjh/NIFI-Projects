# Steps to Add Ldap/AD groups for NIFI Authorisation


## Prerequisites 

### Step 1:  
NIFI must be Secured. Enable SSL and configure node identities and Initial Admin indemnity to login into secured NIFI

Related Doc : https://docs.cloudera.com/HDPDocuments/HDF3/HDF-3.3.0/nifi-authentication/content/nifi_authentication.html


### Step 2 : NIFI must be authenticated. use one of supported authentication mechanism such as Ldap/kerberos ..etc

Related doc : https://docs.cloudera.com/HDPDocuments/HDF3/HDF-3.2.0/nifi-security/content/ldap_login_identity_provider.html






## Steps for enabling Authorisation :

Related Docs :
From Cloudera : https://docs.cloudera.com/HDPDocuments/HDF3/HDF-3.3.0/nifi-security/content/multi-tenant-authorization.html
From Blog : https://pierrevillard.com/2017/12/22/authorizations-with-ldap-synchronization-in-apache-nifi-1-4/



Post making the NIFI cluster secure and authentication enabled, in order to have the NIFI authorisation via LDAP groups and users, we need to perform 2 steps
1. Modify `authorisation.xml` file
2. Update property `nifi.security.user.authorizer`


### Step 1 : modify the authorisation.xml  file 

We Will be implementing composite user groups provider that will provide users with both file-base user group providers where user can manually add users in NIFI UI and also LDAP based user group provider, which will sync the users and groups from LDAP/AD.

==================================================================================================
         
———————————————————————————————————————————————————
  
  
Section 1 : This is the section will be already present in the default xml file in Ambari Ui.  You do not need to change any statement in this section. 

   <authorizers>

            {% if not (has_ranger_admin and enable_ranger_nifi) %}
            <userGroupProvider>
            <identifier>file-user-group-provider</identifier>
            <class>org.apache.nifi.authorization.FileUserGroupProvider</class>
            <property name="Users File">{{nifi_flow_config_dir}}/users.xml</property>
            <property name="Legacy Authorized Users File"></property>
            <property name="Initial User Identity 0">{{nifi_initial_admin_id}}</property>
            {{nifi_ssl_config_content | replace("Node","Initial User")}}
            </userGroupProvider>
———————————————————————————————————————————————————


Section 2 : This is the section added in to the xml file to include the Ldap user group provider. Here there are 2 sub sections . I
 have highlighted the basic properties to be updated in this config in PINK. This config is taken from the example provide in the blog for reference. 
    
    <userGroupProvider>
    <identifier>ldap-user-group-provider</identifier>
    <class>org.apache.nifi.ldap.tenants.LdapUserGroupProvider</class>
    <property name="Authentication Strategy">SIMPLE</property>

    <property name="Manager DN">ldap-manager-dn</property>
    <property name="Manager Password">password</property>

    <property name="Referral Strategy">FOLLOW</property>
    <property name="Connect Timeout">10 secs</property>
    <property name="Read Timeout">10 secs</property>

    <property name="Url">hostname</property>
    <property name="Page Size"></property>
    <property name="Sync Interval">30 mins</property>

    <property name="User Search Base">User search Base</property>
    <property name="User Object Class">person</property>
    <property name="User Search Scope">ONE_LEVEL</property>
    <property name="User Search Filter"></property>
    <property name="User Identity Attribute">cn</property>
    <property name="User Group Name Attribute">title</property>
    <property name="User Group Name Attribute - Referenced Group Attribute"></property>

    <property name="Group Search Base">group search Base</property>
    <property name="Group Object Class">group</property>
    <property name="Group Search Scope">ONE_LEVEL</property>
    <property name="Group Search Filter"></property>
    <property name="Group Name Attribute">cn</property>
    <property name="Group Member Attribute"></property>
    <property name="Group Member Attribute - Referenced User Attribute"></property>
    </userGroupProvider>
———————————————————————————————————————————————————


Section 3 : This is the section added in to the xml file to include the composite user group provider, as explained in the blog

     <userGroupProvider>
        <identifier>composite-configurable-user-group-provider</identifier>
        <class>org.apache.nifi.authorization.CompositeConfigurableUserGroupProvider</class>
         <property name="Configurable User Group Provider">file-user-group-provider</property>
         <property name="User Group Provider 1">ldap-user-group-provider</property>
      </userGroupProvider>
———————————————————————————————————————————————————


Section 4 : This is the section will be already present in the default xml file in amabri Ui.  You need to only update the property highlighted in PINK

            <accessPolicyProvider>
            <identifier>file-access-policy-provider</identifier>
            <class>org.apache.nifi.authorization.FileAccessPolicyProvider</class>
            <property name="User Group Provider">composite-configurable-user-group-provider</property>
            <property name="Authorizations File">{{nifi_flow_config_dir}}/authorizations.xml</property>
            <property name="Initial Admin Identity">{{nifi_initial_admin_id}}</property>
            <property name="Legacy Authorized Users File"></property>
            {{nifi_ssl_config_content}}
            </accessPolicyProvider>
———————————————————————————————————————————————————
           

Section 5 : This is the section will be already present in the default xml file in Ambari Ui.  You need to only update the property highlighted in PINK   

           <authorizer>
            <identifier>managed-authorizer</identifier>
            <class>org.apache.nifi.authorization.StandardManagedAuthorizer</class>
            <property name="Access Policy Provider">file-access-policy-provider</property>
            </authorizer>
            {% else %}
———————————————————————————————————————————————————


Section 6 : This is the section will be already present in the default xml file in Ambari Ui.  You do not need to change any statement in this section. 

            <authorizer>
            <identifier>{{nifi_authorizer}}</identifier>
            <class>org.apache.nifi.ranger.authorization.RangerNiFiAuthorizer</class>
            <property name="Ranger Audit Config Path">{{nifi_config_dir}}/ranger-nifi-audit.xml</property>
            <property name="Ranger Security Config Path">{{nifi_config_dir}}/ranger-nifi-security.xml</property>
            <property name="Ranger Service Type">nifi</property>
            <property name="Ranger Application Id">nifi</property>
            <property name="Ranger Admin Identity">{{ranger_admin_identity}}</property>
            {% if security_enabled %}
            <property name="Ranger Kerberos Enabled">true</property>
            {% else %}
            <property name="Ranger Kerberos Enabled">false</property>
            {% endif %}
            </authorizer>
            {% endif %}

            </authorizers>
———————————————————————————————————————————————————
===============================================================================================



### Step 2  : Update the below property in Ambari to point the authoriser mechanism to use Managed-Authorizer

    nifi.security.user.authorizer=managed-authorizer



- Post the above two steps , you need to restart Nifi services.

Note: you may need to take backup [either rename or move to a different location] of existing Authorization.xml and Users.xml file from location “/var/lib/nifi/conf”, before restart of the Nifi service, for the new users/authorisation changes to be synced, Incase post restart, the changes are not synced or user is facing issue in login to Web UI.



