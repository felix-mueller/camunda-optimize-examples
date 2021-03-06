# Optimize variable plugin - filter out variables that you don't want in Optimize

This example demonstrates how to use an Optimize variable plugin to hook into the
Optimize import and filter out variables you don't want to be added to Optimize. 

After having read through the guide, you will understand

* how the plugin system in Optimize works.
* how to implement your own Optimize variable plugin.
* how to configure the plugin in Optimize.

What is the idea/use case of this demo:

* Variables contain confidential or sensitive data, which you are not allowed to be
analyzed from a legal perspective.
* You don't want to pollute Optimize with unnecessary data.

### How does it work?

To see how the import in Optimize works in general, have a look at the [documentation][4]. 
The details of the variable import and how the plugin hooks into that 
are explained in the [variable import plugin readme][5].

### Setup your environment

First, add the Optimize plugin dependency to your project using mavens [pom.xml][3]:

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

Now with the Optimize dependency added, the minimum requirements are set to
implement your own plugin. For that extend the 
`org.camunda.optimize.plugin.importing.variable.VariableImportAdapter` interface. In 
the given [example][2]:

```java
package org.camunda.optimize.examples;

// imports come here

public class FilterOutSensitiveVariables implements VariableImportAdapter {

  private static final Set<String> sensitiveVariables = Collections.unmodifiableSet(
      new HashSet<>(Arrays.asList("customerName", "customerId"))
  );

  public List<PluginVariableDto> adaptVariables(List<PluginVariableDto> list) {

    return list
      .stream()
      .filter(var -> !sensitiveVariables.contains(var.getName()))
      .collect(Collectors.toList());
  }
}
```

### Build the plugin and add it to your Optimize distribution

Now run the following command to build the Jar containing the implemented plugin:

```cmd
mvn clean install
```

This creates a `filter-out-variables-1.0.0.jar` in `target` folder. Copy this
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

When Optimize is now started, it should automatically filter out variables with the name `customerName`
and `customerId`. All other variables are still imported to Optimize (if they are not complex or 
binary variables).

[2]: src/main/java/org/camunda/optimize/examples/FilterOutSensitiveVariables.java
[3]: pom.xml
[4]: https://docs.camunda.org/optimize/latest/technical-guide/import/import-overview/
[5]: ../README.md

