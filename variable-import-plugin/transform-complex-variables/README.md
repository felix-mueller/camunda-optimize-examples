# Optimize variable plugin - transform complex variables to primitive ones

This example demonstrates how to use an Optimize variable plugin to hook into the
Optimize import and transform complex variables to primitive variables such those
can be easily used in Optimize to analyze them. 

After having read through the guide, you will understand

* how the plugin system in Optimize works.
* how to implement your own Optimize variable plugin.
* how to configure the plugin in Optimize.

What is the idea/use case of this demo:

* You have variables that complex data types (e.g. JSON, XML or java objects) and
still want to analyze certain fields of those variables. 

### How does it work?

To give you a better understanding of how variables are imported in Optimize, 
have a look at the following diagram:

![Variable Import][1]

Using the engine REST-API Optimize fetches all variable, but the variables 
that contain binary data (e.g. pdfs) from the engine. This also includes complex 
variables like JSON, XML oder Java object variables. Then those variables are 
passed through all variable plugins that are configured in Optimize. Here you
can extract the fields of complex variable and add them as primitive variables
to the variable stream. All variables that are still available after the plugins 
are filtered again such that Optimize only imports primitive typed variables. 
Finally, the primitive variables are persisted to Elasticsearch.

### Setup your environment

First, add the Optimize plugin dependency to your project using mavens [pom.xml][4]:

```xml
<dependency>
  <groupId>org.camunda.optimize</groupId>
  <artifactId>plugin</artifactId>
  <version>${optimize.version}</version>
</dependency>
```

To tell maven where to find the plugin environment, add the following repository to your project:

```xml
<repositories>
  <repository>
    <id>camunda-bpm-nexus</id>
    <name>camunda-bpm-nexus</name>
    <url>
      https://app.camunda.com/nexus/content/repositories/camunda-optimize
    </url>
  </repository>
</repositories>
```

### Implement the variable plugin

In this example we assume that you have a business process where you interact with
customers, e.g. handling customer support. Each customer is stored as a JSON variable 
during the process execution. Now you want to analyze how much time you need for the 
support for certain customer and how you can improve the performance of the 
customer support process, which Optimize would be the perfect tool for. Since the 
customer data is stored in a complex JSON variable, Optimize is not able to analyze
it out of the box. However, we can use the Optimize plugin system to hook into the
import and transform each complex customer variable to a set of primitive variables.

For that extend the 
`org.camunda.optimize.plugin.importing.variable.VariableImportAdapter` interface. In 
the given [example][2]:

```java
package org.camunda.optimize.examples;

// imports come here

public class TransformComplexVariables implements VariableImportAdapter {

  private static final ObjectMapper objectMapper = new ObjectMapper();

  @Override
  public List<PluginVariableDto> adaptVariables(List<PluginVariableDto> list) {
    List<PluginVariableDto> resultList = new ArrayList<>();
    for (PluginVariableDto pluginVariableDto : list) {
      if (isCustomerVariable(pluginVariableDto)) {
        extractCustomerFieldsAndAddThemAsVariables(resultList, pluginVariableDto);
      } else {
        resultList.add(pluginVariableDto);
      }
    }
    return resultList;
  }

  private void extractCustomerFieldsAndAddThemAsVariables(List<PluginVariableDto> resultList,
                                                          PluginVariableDto pluginVariableDto) {
    try {
      Customer customer =
        objectMapper.readValue(pluginVariableDto.getValue(), Customer.class);
      resultList.add(createCustomerNameVariable(pluginVariableDto, customer));
      resultList.add(createCustomerIdVariable(pluginVariableDto, customer));
    } catch (IOException e) {
      e.printStackTrace();
    }
  }

  private boolean isCustomerVariable(PluginVariableDto pluginVariableDto) {
    return pluginVariableDto.getName().equals("customer") &&
          pluginVariableDto.getValueInfo().get("objectTypeName").equals("org.camunda.optimize.examples.Customer") &&
      pluginVariableDto.getValueInfo().get("serializationDataFormat").equals("application/json");
  }

  private PluginVariableDto createCustomerNameVariable(PluginVariableDto originalVariable, Customer customer) {
    PluginVariableDto customerNameVar = new PluginVariableDto();
    customerNameVar.setEngineAlias(originalVariable.getEngineAlias());
    customerNameVar.setId(UUID.randomUUID().toString());
    customerNameVar.setProcessDefinitionId(originalVariable.getProcessDefinitionId());
    customerNameVar.setProcessDefinitionKey(originalVariable.getProcessDefinitionKey());
    customerNameVar.setProcessInstanceId(originalVariable.getProcessInstanceId());
    customerNameVar.setVersion(1L);
    customerNameVar.setName("customerName");
    customerNameVar.setType("String");
    customerNameVar.setValue(customer.getName());
    return customerNameVar;
  }

  private PluginVariableDto createCustomerIdVariable(PluginVariableDto originalVariable, Customer customer) {
    PluginVariableDto customerNameVar = new PluginVariableDto();
    customerNameVar.setEngineAlias(originalVariable.getEngineAlias());
    customerNameVar.setId(UUID.randomUUID().toString());
    customerNameVar.setProcessDefinitionId(originalVariable.getProcessDefinitionId());
    customerNameVar.setProcessDefinitionKey(originalVariable.getProcessDefinitionKey());
    customerNameVar.setProcessInstanceId(originalVariable.getProcessInstanceId());
    customerNameVar.setVersion(1L);
    customerNameVar.setName("customerId");
    customerNameVar.setType("String");
    customerNameVar.setValue(customer.getId());
    return customerNameVar;
  }

}
```

The customer class in this example is just a very basic class containing two fields - 
the customer id and the customer name:

```java
public class Customer {

  private String name;
  private String id;

  public String getName() {
    return name;
  }

  public void setName(String name) {
    this.name = name;
  }

  public String getId() {
    return id;
  }

  public void setId(String id) {
    this.id = id;
  }
}
```

### Build the plugin and add it to your Optimize distribution

Now run the following command to build the Jar containing the implemented plugin:

```cmd
mvn clean install
```

This creates a `transform-complex-variables-1.0.0.jar` in `target` folder. Copy this
jar file into the `plugin` folder of your Optimize distribution.

Finally it is necessary to tell Optimize in which package it should search for the plugin. You 
could do that if you add the following lines to `environment-config.yaml` in the 
`environment` folder of your Optimize distribution:
```yaml
plugin:
  variableImport:
    basePackages: ['org.camunda.optimize.examples']
```

The package path must match the one which is written in the implemented java class of the plugin.

When Optimize is now started, it should automatically transform the complex customer variables to
two variables with the name `customerName` and `customerId`. All other variables are still imported 
to Optimize (if they are not complex or binary variables).

[1]: ../../docs/optimize-variable-import.png
[2]: src/main/java/org/camunda/optimize/examples/TransformComplexVariables.java
[3]: src/main/java/org/camunda/optimize/examples/Customer.java
[4]: pom.xml