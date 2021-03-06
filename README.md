# XACML-enabled Authorizer for Apache Kafka

## Terms
* **[XACML](http://docs.oasis-open.org/xacml/3.0/xacml-3.0-core-spec-os-en.html)**: eXtensisble Access Control Markup Language for access policies and access requests/responses, standardized by OASIS.
* **PDP**: Policy Decision Point, as defined in XACML standard.
* **PAP**: Policy Administration Point, as defined in XACML standard.

## Project description
This project provides an [Authorizer](https://kafka.apache.org/documentation/#security_authz) implementation for Apache Kafka that extends the Kafa's default authorizer (`kafka.security.auth.SimpleAclAuthorizer`) to enable getting XACML authorization decisions from a XACML-enabled PDP's REST API as well, according to the [REST Profile of XACML 3.0](http://docs.oasis-open.org/xacml/xacml-rest/v1.0/xacml-rest-v1.0.html). [AuthzForce Server](https://github.com/authzforce/server) and [AuthzForce RESTful PDP](https://github.com/authzforce/restful-pdp) both provide such REST API. Usually, the latter is enough for simple use cases, unless you need a PAP API, multi-tenancy, etc. in which case AuthzForce Server is a better fit (see the [documentation for the full list of features](http://authzforce-ce-fiware.readthedocs.io/en/latest/Features.html))

In other terms, you can still use [Kafka ACLs](http://kafka.apache.org/documentation.html#security_authz) with this same authorizer as you would with the default one. XACML evaluation must be enabled explicitly by setting specific properties as described later below. *XACML evaluation* here stands for the extra process of getting a XACML authorization decision from a remote PDP according to the REST Profile of XACML 3.0.

The authorizer combines Kafka ACL evaluation with XACML evaluation as follows: 

* If ACL evaluation returns Permit, return Permit.
* Else: 
  * If XACML evaluation is disabled, return Deny.
  * Else: If and only if the result of XACML evaluation is Permit, return Permit.
  
## Installation
Get the `tar.gz` distribution from the [latest release on the GitHub repository](https://github.com/DRIVER-EU/kafka-combined-acl-xacml-authorizer/releases) and extract the files to some folder, e.g. `/opt/authzforce-ce-kafka-extensions`. You should have a `lib` folder inside.

## Configuration
To enable the authorizer on Kafka, set the server's property: 

`authorizer.class.name=org.ow2.authzforce.kafka.pep.CombinedXacmlAclAuthorizer`

To enable XACML evaluation, set the extra following authorizer properties:
* **`org.ow2.authzforce.kafka.pep.xacml.pdp.url`**: XACML PDP resource's URL, as defined by [REST Profile of XACML 3.0](http://docs.oasis-open.org/xacml/xacml-rest/v1.0/xacml-rest-v1.0.html), §2.2.2, e.g. `https://serverhostname/services/pdp` for a [AuthzForce RESTful PDP](https://github.com/authzforce/restful-pdp) instance, or `https://serverhostname/authzforce-ce/domains/XXX/pdp` for a domain `XXX` on a [AuthzForce Server](https://github.com/authzforce/server) instance.
* **`org.ow2.authzforce.kafka.pep.http.client.cfg.location`**: location (URL supported by Spring {@link org.springframework.util.ResourceUtils}) of the HTTP client configuration as defined by <a href="https://cxf.apache.org/docs/client-http-transport-including-ssl-support.html#ClientHTTPTransport(includingSSLsupport)-UsingConfiguration">Apache CXF format</a>, required for SSL settings
* **`org.ow2.authzforce.kafka.pep.authz.cache.size.max`:** maximum number of authorization decisions cached in memory (performance optimization). Cache disabled iff not strictly positive integer. If cache enabled and an access request matches a previous one in cache, the corresponding decision is retrieved from cache directly (no decision evaluation).
* **`org.ow2.authzforce.kafka.pep.xacml.req.tmpl.location`:** location of a file that contains a [Freemarker](https://freemarker.apache.org/) template of XACML Request formatted according to [JSON Profile of XACML 3.0](http://docs.oasis-open.org/xacml/xacml-json-http/v1.0/xacml-json-http-v1.0.html), in which you can use [Freemarker expressions](https://freemarker.apache.org/docs/dgui_template_exp.html), enclosed between `${` and `}`, and have access to the following [top-level variables](https://freemarker.apache.org/docs/dgui_template_exp.html#dgui_template_exp_var_toplevel) from Kafka's authorization context:

| Variable name | Variable type | Description |
| --- | --- | --- |
|`clientHost` | [java.net.InetAddress](https://docs.oracle.com/javase/8/docs/api/java/net/InetAddress.html) | client/user host name or IP address |
|`principal`| [org.apache.kafka.common.security.auth.KafkaPrincipal](https://kafka.apache.org/11/javadoc/org/apache/kafka/common/security/auth/KafkaPrincipal.html)| user principal|
|`operation`|[org.apache.kafka.common.acl.AclOperation](http://kafka.apache.org/11/javadoc/index.html?org/apache/kafka/common/acl/AclOperation.html)|operation|
|`resourceType`|[org.apache.kafka.common.resource.ResourceType](https://kafka.apache.org/11/javadoc/org/apache/kafka/common/resource/ResourceType.html)|resource type|
|`resourceName`|`String`|resource name|

For an example of XACML Request template, see the file `request.xacml.json.ftl` in the [source](src/test/resources/request.xacml.json.ftl) or in the same folder as this README if part of a release package (tar.gz). This example should be sufficient for most cases.

## Starting Kafka
Make sure Zookeeper is started first:
```sh
~/DRIVER+/kafka_2.11-1.1.0$ bin/zookeeper-server-start.sh config/zookeeper.properties
```

Add the all JARs in the `lib` folder extracted earlier (*Installation* section) to the CLASSPATH environment variable before starting Kafka, for example:

```sh
~/DRIVER+/kafka_2.11-1.1.0$ CLASSPATH=/opt/authzforce-ce-kafka-extensions/lib/* bin/kafka-server-start.sh config/server.properties
```
