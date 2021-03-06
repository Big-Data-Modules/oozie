

[::Go back to Oozie Documentation Index::](index.html)

# Action Authentication

<!-- MACRO{toc|fromDepth=1|toDepth=4} -->

## Background

A secure cluster requires that actions have been authenticated (typically via Kerberos).  However, due to the way that Oozie runs
actions, Kerberos credentials are not easily made available to actions launched by Oozie.  For many action types, this is not a
problem because they are self contained (beyond core Hadoop components).  For example, a Pig action typically only talks to
MapReduce and HDFS.  However, some actions require talking to external services (e.g. HCatalog, HBase Region Server, Hive Server 2)
and in these cases, the actions require some extra configuration in Oozie to authenticate.  To be clear, this extra configuration
is only required if an action will be talking to these types of external services; running a typical MapReduce, Pig, Hive, etc
action will not require any of this.

For these situations, Oozie will have to use its Kerberos credentials to obtain "delegation tokens" (think of it like a cookie) on
behalf of the user from the service in question.  The details of what this means is beyond the scope of this documentation, but
basically, Oozie needs some extra configuration in the workflow so that it can obtain this delegation token.

## Oozie Server Configuration

The code to obtain delegation tokens is pluggable so that it is easy to add support for different services by simply subclassing
org.apache.oozie.action.hadoop.Credentials to retrieve a delegation token from the service and add it to the Configuration.

Out of the box, Oozie already comes with support for some credential types
(see [Built-in Credentials Implementations](DG_ActionAuthentication.html#Built-in_Credentials_Implementations)).
The credential classes that Oozie should load are specified by the following property in oozie-site.xml.  The left hand side of the
equals sign is the type for the credential type, while the right hand side is the class.


```
   <property>
      <name>oozie.credentials.credentialclasses</name>
      <value>
         hcat=org.apache.oozie.action.hadoop.HCatCredentials,
         hbase=org.apache.oozie.action.hadoop.HbaseCredentials,
         hive2=org.apache.oozie.action.hadoop.Hive2Credentials
      </value>
   </property>
```

## Workflow Changes

The user should add a `credentials` section to the top of their workflow that contains 1 or more `credential` sections.  Each of
these `credential` sections contains a name for the credential, the type for the credential, and any configuration properties
needed by that type of credential for obtaining a delegation token.  The `credentials` section is available in workflow schema
version 0.3 and later.

For example, the following workflow is configured to obtain an HCatalog delegation token, which is given to a Pig action so that the
Pig action can talk to a secure HCatalog:


```
   <workflow-app xmlns='uri:oozie:workflow:0.4' name='pig-wf'>
      <credentials>
         <credential name='my-hcat-creds' type='hcat'>
            <property>
               <name>hcat.metastore.uri</name>
               <value>HCAT_URI</value>
            </property>
            <property>
               <name>hcat.metastore.principal</name>
               <value>HCAT_PRINCIPAL</value>
            </property>
         </credential>
      </credentials>
      ...
      <action name='pig' cred='my-hcat-creds'>
         <pig>
            <job-tracker>JT</job-tracker>
            <name-node>NN</name-node>
            <configuration>
               <property>
                  <name>TESTING</name>
                  <value>${start}</value>
               </property>
            </configuration>
         </pig>
      </action>
      ...
   </workflow-app>
```

The type of the `credential` is "hcat", which is the type name we gave for the HCatCredentials class in oozie-site.xml.  We gave
the `credential` a name, "my-hcat-creds", which can be whatever you want; we then specify cred='my-hcat-creds' in the Pig action,
so that Oozie will include these credentials with the action.  You can include multiple credentials with an action by specifying
a comma-separated list of `credential` names.  And finally, the HCatCredentials required two properties (the metastore URI and
principal), which we also specified.

Adding the `credentials` section to a workflow and referencing it in an action will make Oozie always try to obtain that delegation
token.  Ordinarily, this would mean that you cannot re-use this workflow in a non-secure cluster without editing it because trying
to obtain the delegation token will likely fail.  However, you can tell Oozie to ignore the `credentials` for a workflow by setting
the job-level property `oozie.credentials.skip` to `true`; this will allow you to use the same workflow.xml in a secure and
non-secure cluster by simply changing the job-level property at runtime. If omitted or set to `false`, Oozie will handle
the `credentials` section normally. In addition, you can also set this property at the action-level or server-level to skip getting
credentials for just that action or for all workflows, respectively.  The order of priority is this:

   1. `oozie.credentials.skip` in the `configuration` section of an action, if set
   1. `oozie.credentials.skip` in the job.properties for a workflow, if set
   1. `oozie.credentials.skip` in oozie-site.xml for all workflows, if set
   1. (don't skip)

## Built-in Credentials Implementations

Oozie currently comes with the following Credentials implementations:

   1. HCatalog and Hive Metastore: `org.apache.oozie.action.hadoop.HCatCredentials`
   1. HBase: `org.apache.oozie.action.hadoop.HBaseCredentials`
   1. Hive Server 2: `org.apache.oozie.action.hadoop.Hive2Credentials`
   1. File system (for workflows that require cross cluster or cloud storage access):
   `org.apache.oozie.action.hadoop.FileSystemCredentials`

HCatCredentials requires these two properties:

   1. `hcat.metastore.principal` or hive.metastore.kerberos.principal
   1. `hcat.metastore.uri` or hive.metastore.uris

**Note:** The HCatalog Metastore and Hive Metastore are one and the same and so the "hcat" type credential can also be used to talk
to a secure Hive Metastore, though the property names would still start with "hcat.".

HBase does not require any additional properties since the hbase-site.xml on the Oozie server provides necessary information
to obtain a delegation token; though properties can be overwritten here if desired.

Hive2Credentials requires these two properties:
   1. `hive2.server.principal`
   1. `hive2.jdbc.url`

FileSystemCredentials requires the following property:

   1. `filesystem.path` - Cloud storage bucket or namenode path, where the action will need access rights in runtime.

[::Go back to Oozie Documentation Index::](index.html)


